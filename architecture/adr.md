# Architectural Decision Records

| ADR | Decision | Rationale |
|---|---|---|
| **ADR-1** | Support BOTH MTProto and Bot API bidirectionally | Different use cases require different protocols. Parsing needs MTProto; user interaction needs Bot API. Both send and receive. |
| **ADR-2** | Remote PostgreSQL (not local/embedded) as Go-side data store | Proper hosted database for durability, backups, connection pooling. Not bbolt/SQLite — we need concurrent writes, queue processing, and multi-table relationships. |
| **ADR-3** | ~~Cloudflare R2 + Workers for media~~ **SUPERSEDED by ADR-20** | ~~Salesforce storage is expensive (~2 GB/user). R2 has zero egress fees.~~ ContentVersion is simpler and sufficient for MVP. See ADR-20. |
| **ADR-4** | ~~Centrifugo for real-time LWC chat~~ **SUPERSEDED by ADR-21** | ~~Salesforce can't host WebSockets.~~ Platform Events via empApi are sufficient for MVP scale. See ADR-21. |
| **ADR-5** | Single corporate API ID + residential proxy pool | Per-user API IDs violate Telegram ToS. Shared API ID from single IP triggers bans. Proxy isolation per session is the proven strategy. |
| **ADR-6** | HMAC-SHA256 for webhook auth | Simpler than OAuth for high-velocity stateless pushes. Constant-time comparison prevents timing attacks. Pre-shared secret in Protected Custom Metadata. |
| **ADR-7** | Hybrid ingestion: Platform Events + Bulk API 2.0 + REST | No single API handles all cases. Events for live UI, Bulk for archives, REST for ad-hoc single updates. |
| **ADR-8** | Pub/Sub API (gRPC) for Go <-> Salesforce event streaming | Replaces legacy CometD. ~7-10x faster (compressed protobuf). Bidirectional streaming, pull-based flow control. Excellent Go gRPC support. |
| **ADR-9** | Hetzner (Europe) as primary hosting | Proximity to Telegram DCs in Amsterdam. Clean IPs. ARM Ampere instances available. Cost-effective for long-running Go processes. |
| **ADR-10** | Messenger-agnostic Go interface design | MVP is Telegram, but WhatsApp/Messenger/Viber planned. Adapter pattern: new platforms = new Go files, not restructuring core. |
| **ADR-11** | 2GP Managed Package (second-generation) | Salesforce recommends 2GP for all new packages. Better CI/CD integration, scratch org support, semantic versioning. |
| **ADR-12** | Generic `Messenger_*` naming for all objects/tables | Not `Telegram_*`. Supports multi-platform from day one without schema migrations when adding new platforms. |
| **ADR-13** | ~~Centrifugo JWT auth via Apex~~ **SUPERSEDED by ADR-21** | ~~Centrifugo JWT auth.~~ Centrifugo removed. See ADR-21. |
| **ADR-14** | ~~Direct-to-R2 outbound media upload via SigV4 presigned URLs~~ **SUPERSEDED by ADR-20** | ~~Avoids routing binary files through Salesforce storage tier.~~ ContentVersion REST API (multipart) replaces R2 presigned URLs. See ADR-20. |
| **ADR-15** | SRP v6a implementation for MTProto 2FA via gotd/td crypto/srp | PBKDF2 with 100K iterations must run on Go backend (browser JS would lock UI). Zero-knowledge proof — password never transmitted in plaintext. |
| **ADR-16** | Platform Events for outbound delivery failure sync | Real-time UI feedback for agents (red exclamation mark on failed messages) + durable persistence via dual-consumer pattern (LWC empApi + Apex Trigger). |
| **ADR-17** | sync.Pool + streaming JSON encoder for batch byte tracking | Prevents GC thrashing under high-throughput bursts. Individual struct-level byte tracking avoids full-slice re-serialization. 950KB threshold with 2-3 sec sliding window. |
| **ADR-18** | UNLOGGED tables + periodic LOGGED backup for session durability | UNLOGGED gives 10-50x write performance for MTProto session updates. Periodic snapshot (every 5 min) to a LOGGED backup table provides crash recovery with bounded data loss. Maximum loss: 5 minutes of session state — users see brief reconnection, not full re-auth. |
| **ADR-19** | Data Flow Routing Matrix — clear API assignment per data flow | Eliminates overlap between ADR-7 (hybrid ingestion) and ADR-8 (Pub/Sub API). See routing table below. |
| **ADR-20** | Salesforce ContentVersion for all media (supersedes ADR-3, ADR-14) | R2 eliminated. ContentVersion is simpler, avoids external infrastructure, sufficient for MVP file storage needs. Media accessed via `/sfc/servlet.shepherd/` URLs with session-based auth. |
| **ADR-21** | Platform Events (empApi) for real-time UI (supersedes ADR-4, ADR-13) | Centrifugo eliminated. No AppExchange competitor uses external WebSocket. Platform Events via empApi are sufficient for MVP scale. |

---

## ADR-18: Session Durability Strategy

**Status:** Accepted
**Date:** 2026-03-29
**Context:** MTProto sessions are stored in PostgreSQL UNLOGGED tables (ADR-2) for write performance. UNLOGGED tables skip WAL — if the server crashes or PostgreSQL restarts, all session data is lost. Every connected user must re-authenticate (phone + OTP + 2FA).

**Decision:** Keep UNLOGGED tables for session writes. Add a periodic snapshot mechanism that copies `mtproto_sessions` to `mtproto_sessions_backup` (a regular LOGGED table) every 5 minutes. On startup, if `mtproto_sessions` is empty but `mtproto_sessions_backup` has data, restore from backup.

**Consequences:**
- Maximum data loss on crash: 5 minutes of session state
- Users see brief reconnection (seconds), not full re-authentication (minutes)
- Write performance remains 10-50x faster than LOGGED for session updates
- Backup snapshot adds ~1 query every 5 minutes (negligible overhead)
- Requires `002_session_backup.sql` migration for the backup table

**Alternatives rejected:**
- Full WAL (LOGGED tables): ~2-3x slower writes, unacceptable for high-frequency session heartbeats
- Accept total loss: At 100+ sessions, mass re-auth creates support burden and poor UX
- External session store (Redis): Adds operational complexity without clear benefit over PostgreSQL

---

## ADR-19: Data Flow Routing Matrix

**Status:** Accepted
**Date:** 2026-03-29
**Context:** ADR-7 defines hybrid ingestion (Platform Events + Bulk API + REST). ADR-8 introduces Pub/Sub API (gRPC). Their roles overlap without a clear routing table, causing developer confusion about which API to use for each data flow.

**Decision:** Assign a primary API to each data flow:

| Data Flow | Direction | API | Rationale |
|---|---|---|---|
| New inbound messages (real-time) | Go → SF | Platform Events (`Inbound_Message__e`) | Low latency, trigger-driven record creation |
| Session status changes | Go → SF | Platform Events (`Session_Status__e`) | Infrequent, critical for UI state |
| Delivery status updates | Go → SF | Platform Events (`Message_Delivery_Status__e`) | Real-time UI feedback via dual-consumer |
| Outbound message requests | SF → Go | REST API (from Queueable Apex) | Request-response, single message |
| Bulk message archive | Go → SF | Bulk API 2.0 | High volume, async acceptable |
| Channel config read at startup | Go ← SF | REST API (SOQL query) | One-time read, small payload |
| High-throughput streaming (future) | Go ↔ SF | Pub/Sub API (gRPC) | When PE 50K/24h delivery limit is hit |

**Consequences:**
- Clear ownership: each data flow has exactly one primary API
- Pub/Sub API is explicitly future-state, not MVP
- Platform Events remain the workhorse for real-time Go→SF communication
- REST API handles all SF→Go callouts (outbound messages, config reads)
- Bulk API 2.0 reserved for batch operations only

**Alternatives rejected:**
- Pub/Sub API for everything: Overkill for MVP, adds gRPC complexity
- REST API for everything: No trigger-driven processing, higher latency for inbound

---

## ADR-20: Remove Cloudflare R2 — Use Salesforce ContentVersion for All Media

**Status:** Accepted (supersedes ADR-3 and ADR-14)
**Date:** 2026-04-01
**Context:** ADR-3 chose Cloudflare R2 + Workers for media storage, citing Salesforce storage costs (~2 GB/user) and zero egress fees. ADR-14 defined SigV4 presigned URLs for direct-to-R2 uploads from LWC.

After architectural review:
1. **ContentVersion is sufficient for MVP scale.** File storage quotas (~2 GB/user) are adequate for early customers with moderate media volume.
2. **R2 adds significant infrastructure complexity:** separate bucket provisioning, Worker deployment, CORS configuration, SigV4 presigned URL generation, HMAC media auth, and a `media_files` PostgreSQL tracking table.
3. **AppExchange security review is simpler** without external storage dependencies.
4. **Native Salesforce media access** via `/sfc/servlet.shepherd/` URLs uses session-based authentication — no custom auth layer needed.

**Decision:** Remove Cloudflare R2 and Workers entirely. Store all media as Salesforce ContentVersion records. Link files to Message/Attachment records via ContentDocumentLink.

**Media architecture:**
```
Go Middleware --ContentVersion REST API (multipart)--> Salesforce Files
LWC (browser) --/sfc/servlet.shepherd/ URL--> ContentVersion (session-based access)
```

**Consequences:**
- Eliminates Cloudflare R2 bucket, Workers, CORS configuration
- Eliminates `media_files` PostgreSQL table
- Eliminates SigV4 presigned URL generation
- Eliminates HMAC media display authentication
- Removes R2 env vars (`R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`)
- Simplifies AppExchange security review (no external storage dependency)
- Consumes Salesforce file storage quota (acceptable at MVP scale)
- If storage becomes a constraint at scale, R2 or external storage can be re-introduced as an optional tier

**Alternatives rejected:**
- Keep R2 for MVP: Over-engineering. Adds infrastructure complexity for a problem that doesn't exist at MVP scale
- Amazon S3: Same complexity as R2, with egress fees

---

## ADR-21: Remove Centrifugo — Use Native SF Platform Events for Real-Time UI

**Status:** Accepted (supersedes ADR-4 and ADR-13)
**Date:** 2026-04-01
**Context:** ADR-4 chose Centrifugo (external WebSocket server) for real-time LWC chat updates, citing Platform Event delivery limits (50K/24h). ADR-13 defined JWT auth for Centrifugo. Competitor research (see `research/competitor-realtime-architecture.md`) revealed:

1. **No AppExchange messaging app uses an external WebSocket server.** SMS-Magic and Mogli use PushTopics/empApi. Heymarket polls every 60s. All have paying customers at scale.
2. **Platform Event limits are only a problem at scale.** At MVP scale (500-2K messages/day, 3-10 agents), PE daily deliveries stay well within limits.
3. **Centrifugo adds significant complexity:** extra infrastructure, CSP Trusted Sites per customer, JS client bundling, JWT auth from Apex, AppExchange security review burden, and a single point of failure requiring a fallback mechanism anyway.
4. **300ms (CometD) vs 50ms (WebSocket) latency is imperceptible** in a chat application.

**Decision:** Remove Centrifugo entirely. Use Platform Events via `lightning/empApi` in LWC for real-time UI updates. The Go middleware publishes Platform Events to Salesforce; LWC subscribes via empApi.

**Real-time architecture:**
```
Go Middleware --Platform Event--> Salesforce Event Bus --empApi/CometD--> LWC (browser)
```

**For the Telegram auth flow (OTP/2FA):** Replace WebSocket with short-polling from LWC via Apex callout to Go middleware. Auth sessions are short-lived (< 2 minutes), so polling every 1-2 seconds is acceptable.

**Scaling strategy (future, NOT MVP):**
- If a customer exceeds PE delivery limits, evaluate options at that time:
  - High-Volume Platform Event add-on license
  - Pub/Sub API (gRPC) as already planned in ADR-8/ADR-19
  - External WebSocket as opt-in premium tier (re-evaluate if needed)
- Do not over-engineer for hypothetical scale before having paying customers

**Consequences:**
- Eliminates `MessageForge.Centrifugo/` directory entirely
- Removes Centrifugo env vars (`CENTRIFUGO_URL`, `CENTRIFUGO_API_KEY`, `CENTRIFUGO_SECRET`)
- Removes `CentrifugoTokenController.cls` from Apex
- Removes CSP Trusted Sites requirement for WebSocket domain
- Removes `realtime/` package from Go middleware
- Simplifies AppExchange security review (no external WebSocket dependency)
- Simplifies customer setup (no CSP configuration needed)
- Consumes Platform Event delivery allocations (acceptable at MVP scale)

**Alternatives rejected:**
- Keep Centrifugo for MVP: Over-engineering. Adds 3+ weeks of work for imperceptible UX benefit at MVP scale
- Polling only (no Platform Events): Heymarket proves polling works, but empApi is trivial to implement and provides genuine push semantics
