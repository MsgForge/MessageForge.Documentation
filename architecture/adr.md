# Architectural Decision Records

| ADR | Decision | Rationale |
|---|---|---|
| **ADR-1** | Support BOTH MTProto and Bot API bidirectionally | Different use cases require different protocols. Parsing needs MTProto; user interaction needs Bot API. Both send and receive. |
| **ADR-2** | Remote PostgreSQL (not local/embedded) as Go-side data store | Proper hosted database for durability, backups, connection pooling. Not bbolt/SQLite — we need concurrent writes, queue processing, and multi-table relationships. |
| **ADR-3** | Cloudflare R2 + Workers for media | Salesforce storage is expensive (~2 GB/user). R2 has zero egress fees. Worker-based HMAC auth is more secure than pre-signed URLs for display. |
| **ADR-4** | Centrifugo for real-time LWC chat | Salesforce can't host WebSockets. Platform Event limits (50K delivery/24h) too restrictive for active chat. Centrifugo is Go-native, deploys alongside middleware. |
| **ADR-5** | Single corporate API ID + residential proxy pool | Per-user API IDs violate Telegram ToS. Shared API ID from single IP triggers bans. Proxy isolation per session is the proven strategy. |
| **ADR-6** | HMAC-SHA256 for webhook auth | Simpler than OAuth for high-velocity stateless pushes. Constant-time comparison prevents timing attacks. Pre-shared secret in Protected Custom Metadata. |
| **ADR-7** | Hybrid ingestion: Platform Events + Bulk API 2.0 + REST | No single API handles all cases. Events for live UI, Bulk for archives, REST for ad-hoc single updates. |
| **ADR-8** | Pub/Sub API (gRPC) for Go <-> Salesforce event streaming | Replaces legacy CometD. ~7-10x faster (compressed protobuf). Bidirectional streaming, pull-based flow control. Excellent Go gRPC support. |
| **ADR-9** | Hetzner (Europe) as primary hosting | Proximity to Telegram DCs in Amsterdam. Clean IPs. ARM Ampere instances available. Cost-effective for long-running Go processes. |
| **ADR-10** | Messenger-agnostic Go interface design | MVP is Telegram, but WhatsApp/Messenger/Viber planned. Adapter pattern: new platforms = new Go files, not restructuring core. |
| **ADR-11** | 2GP Managed Package (second-generation) | Salesforce recommends 2GP for all new packages. Better CI/CD integration, scratch org support, semantic versioning. |
| **ADR-12** | Generic `Messenger_*` naming for all objects/tables | Not `Telegram_*`. Supports multi-platform from day one without schema migrations when adding new platforms. |
| **ADR-13** | Centrifugo JWT auth via Apex-generated tokens | Centrifugo is stateless — needs cryptographic trust from Salesforce session context. HMAC-SHA256 default (simpler), RSA-SHA256 for cross-org deployments. Short-lived JWTs (15 min) with LWC refresh callback. |
| **ADR-14** | Direct-to-R2 outbound media upload via SigV4 presigned URLs | Avoids routing binary files through Salesforce storage tier (2 GB quota, DB bloat). LWC streams to R2 with near-zero memory footprint via Blob interface. |
| **ADR-15** | SRP v6a implementation for MTProto 2FA via gotd/td crypto/srp | PBKDF2 with 100K iterations must run on Go backend (browser JS would lock UI). Zero-knowledge proof — password never transmitted in plaintext. |
| **ADR-16** | Platform Events for outbound delivery failure sync | Real-time UI feedback for agents (red exclamation mark on failed messages) + durable persistence via dual-consumer pattern (LWC empApi + Apex Trigger). |
| **ADR-17** | sync.Pool + streaming JSON encoder for batch byte tracking | Prevents GC thrashing under high-throughput bursts. Individual struct-level byte tracking avoids full-slice re-serialization. 950KB threshold with 2-3 sec sliding window. |
| **ADR-18** | UNLOGGED tables + periodic LOGGED backup for session durability | UNLOGGED gives 10-50x write performance for MTProto session updates. Periodic snapshot (every 5 min) to a LOGGED backup table provides crash recovery with bounded data loss. Maximum loss: 5 minutes of session state — users see brief reconnection, not full re-auth. |
| **ADR-19** | Data Flow Routing Matrix — clear API assignment per data flow | Eliminates overlap between ADR-7 (hybrid ingestion) and ADR-8 (Pub/Sub API). See routing table below. |

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
