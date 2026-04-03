> **Note:** This review predates ADR-20 (ContentVersion, 2026-03-30) and ADR-21 (Platform Events, 2026-04-01). References to R2, Centrifugo, and Cloudflare Workers reflect the architecture as of 2026-03-29 and are no longer current.

# MessageForge Architecture Review — 2026-03-29

**Reviewer:** Claude Opus 4.6 (automated deep review)
**Scope:** Full project — Documentation, Go Backend, Salesforce Package, PoC, Infrastructure, Claude Config
**Verdict:** IMPLEMENTABLE with 12 critical issues requiring resolution before production

---

## Executive Summary

MessageForge is a multi-messenger to Salesforce CRM integration platform. The architecture is **well-designed, internally consistent, and implementable**. However, significant gaps exist between documentation (which is excellent) and actual implementation (~40% complete across all sub-projects).

| Dimension | Score | Detail |
|---|---|---|
| Architecture Design | 9/10 | ADRs thorough, patterns well-chosen, multi-platform ready |
| Documentation Quality | 9/10 | Comprehensive, consistent, actionable |
| Implementation Completeness | 38% | SF: 8/21 objects. Go: pipeline stubs, nil dependencies |
| Security Posture | 6/10 | Good design, 7 unchecked items, no DAST, unprotected CMT |
| Production Readiness | 2/10 | No deployment guide, no HA, no monitoring, no cost model |
| Agent/Tooling Readiness | 8/10 | 4 agents, 9 skills, 6 commands — mature Claude Code setup |

**Bottom line:** The documentation describes a production-grade system. The code is MVP scaffolding. The gap is bridgeable — all critical design decisions are made and documented. What remains is disciplined execution.

---

## 1. Can This Be Implemented? YES — With Caveats

### What Works Well

1. **Architecture is sound.** The three-layer model (Channel -> Config -> App), adapter pattern for multi-platform, and Go middleware as protocol bridge are all correct choices.

2. **ADRs cover the hard decisions.** 17 ADRs with clear rationale. Key wins:
   - ADR-4: ~~Centrifugo over Platform Events for live chat~~ **SUPERSEDED by ADR-21:** Platform Events (empApi) now used for real-time LWC updates
   - ADR-10: Messenger-agnostic interfaces (avoids restructuring for WhatsApp)
   - ADR-14: ~~Direct-to-R2 presigned URLs~~ **SUPERSEDED by ADR-20:** All media now stored in Salesforce ContentVersion
   - ADR-17: sync.Pool + streaming JSON (prevents GC thrashing at scale)

3. **Research is comprehensive.** 4 research documents (1,200+ lines of technical specification) cover every edge case: SRP v6a crypto, Centrifugo JWT gotchas, R2 CORS bilateral config, Platform Event delivery math.

4. **PoC validates core assumption.** Telegram MTProto + Bot API dual-protocol works in Go with `gotd/td`. Auth flow (OTP + 2FA) is proven.

5. **Go patterns are clean.** oklog/run lifecycle, local interfaces, constructor-at-end, TDD approach — the rewritten backend follows excellent Go idioms.

### What Needs Work Before Implementation Can Succeed

See Section 3 (Critical Issues) and the companion [Risk Register](./2026-03-29-risk-register.md).

---

## 2. Implementation Status — Gap Analysis

### Salesforce Package (38% complete)

| Category | Documented | Implemented | Gap |
|---|---|---|---|
| Custom Objects (non-CMT) | 18 | 4 | 14 missing |
| Custom Metadata Types | 3 | 1 | 2 missing (Protected CMTs) |
| Platform Events | 3 | 3 | Complete |
| Apex Classes | 8+ | 8 | Structurally complete, needs v11 alignment |
| Apex Triggers | 5 | 3 | 2 missing (ChannelAccessTrigger, ChannelCompromiseTrigger) |
| LWC Components | 5 | 4 | 1 missing (channelSetupWizard) |
| Permission Sets | 3 | 2 | Messenger_Viewer missing |

**Critical:** The actual code uses `Messenger_Connection__c` (6 fields) instead of the documented `Messenger_Channel__c` (13+ fields). This is a **naming and schema mismatch** — the v11 data model describes a completely different object hierarchy than what exists in code.

**Objects NOT created (13):**
- All 5 App objects (Telegram_App, WhatsApp_App, Twilio_App, Vonage_App, Bird_App)
- All 8 Config objects (Telegram_Bot_Config, Telegram_User_Config, WhatsApp_Config, Viber_Config, Twilio_Config, Vonage_Config, Bird_Config, Apple_Messages_Config)
- Channel_User_Access__c (access control)
- Messenger_Template__c (WhatsApp templates)
- Messenger_Attachment__c (multi-media)
- Channel_Audit_Log__c (audit trail)
- Channel_Type__mdt (reference data)

### Go Backend (40-50% complete)

| Component | Status | Notes |
|---|---|---|
| HTTP Server + HMAC | Working | Health endpoint, webhook auth |
| Telegram Bot API Handler | Working | Message parsing, media detection |
| Rate Limiting | Working | Per-bot + per-chat token buckets |
| PostgreSQL Pool | Code exists | Not wired into app (nil in NewApp) |
| Database Migrations | Code exists | Not executed from main |
| Salesforce JWT Auth | Working | RS256, auto-refresh |
| Salesforce Batch Ingestion | Working | sync.Pool, 950KB threshold |
| Centrifugo Publisher | Working | gocent v3, channel naming |
| R2 Media Client | Working | Upload, download, presigned URLs |
| Inbound Pipeline | Partial | Fan-out architecture, media=nil, ingester=nil |
| Outbound Pipeline | Stub | Handler exists, not wired |
| Delivery Status Sync | Stub | File exists, no implementation |
| MTProto Client | PoC only | Not in Backend, only in MessageForge.PoC |
| Queue Workers | Not started | Schema exists, no worker goroutines |

**Critical:** `NewApp()` creates the pipeline with `nil` for media processor and message ingester. The architecture is correct but the wiring is incomplete.

### PoC (90% complete)

Standalone demo works: configure API keys -> authenticate -> browse chats -> send messages. In-memory only, no persistence. Validates core Telegram integration.

### Centrifugo (docs only)

Only `CLAUDE.md` exists. No docker-compose, no deployment config, no actual Centrifugo server setup (the docker-compose in Backend is the closest).

### Admin Dashboard (empty)

Only `CLAUDE.md` placeholder. No code, no design, no spec.

---

## 3. Critical Issues (12)

### CRITICAL — Must fix before any production deployment

#### C1: Messenger_Connection__c vs Messenger_Channel__c Schema Mismatch

**What:** Actual Salesforce code uses `Messenger_Connection__c` (6 fields, Telegram-only). Documentation specifies `Messenger_Channel__c` (13+ fields, platform-agnostic) with 8 Config + 5 App child objects.

**Impact:** Every Apex class, trigger, LWC, and Go client references the wrong object API name. Migration requires: creating new objects, migrating data, updating all code references.

**Decision needed:** Either rename `Messenger_Connection__c` to `Messenger_Channel__c` and add fields, OR update all documentation to match current naming. The v11 data model is clearly the target — implement it.

#### C2: Middleware_Config__mdt is NOT Protected

**What:** `Middleware_Config__mdt` stores HMAC secrets and Centrifugo tokens. It is a regular Custom Metadata Type, not Protected. Any admin or developer in the customer org can query `Middleware_Config__mdt.getInstance('Default')` and read all secrets. (**Note (ADR-21):** Centrifugo tokens no longer needed; Centrifugo eliminated in favor of Platform Events.)

**Impact:** AppExchange security review will FAIL. Credential exposure to customer org users.

**Fix:** Create `tgint__Encryption_Key__mdt` (Protected) as documented in v11. Migrate HMAC_Secret and Centrifugo_HMAC_Secret to Protected CMT. Delete `Middleware_Config__mdt`.

#### C3: No DAST Scan — AppExchange Blocker

**What:** Security checklist shows `[ ] DAST scan reports`. No penetration testing has been done on the Go middleware.

**Impact:** AppExchange requires zero high-severity vulnerabilities from OWASP ZAP, Burp Suite, or Chimera. This is a hard blocker for publication.

**Action:** Set up OWASP ZAP against a test deployment of the Go middleware. Fix any findings. Generate report.

#### C4: UNLOGGED MTProto Sessions — No Recovery Strategy

**What:** PostgreSQL UNLOGGED tables skip WAL for performance. If the server crashes or PostgreSQL restarts, ALL session data is lost. Every connected user must re-authenticate (phone + OTP + 2FA).

**Impact:** At scale (100+ active sessions), a single crash creates a mass re-authentication event. Users see "disconnected" with no explanation.

**Missing:** No ADR for this trade-off. No recovery strategy documented. No user notification mechanism.

**Options:**
- A) Keep UNLOGGED but implement graceful degradation (auto-reconnect, notify SF, queue re-auth)
- B) Switch to LOGGED tables with periodic CHECKPOINT (slower writes but crash-safe)
- C) Hybrid: UNLOGGED + periodic snapshot to LOGGED backup table

#### C5: Single Point of Failure — No HA Design

**What:** Architecture shows one Go middleware instance. No load balancing, failover, clustering, or health monitoring documented.

**Impact:** Go process crash = all messaging stops. No redundancy.

**Missing:** Deployment guide, HA architecture, health check integration, auto-restart mechanism. (**Note:** ADR-20 and ADR-21 simplified deployment — no R2, Workers, or Centrifugo to deploy.)

#### C6: Residential Proxy Pool — Unchecked and Uncosted

**What:** ADR-5 requires residential proxy pools for MTProto IP isolation. Security checklist shows `[ ] Residential proxy pool` (not done). No cost analysis, no provider selected, no rotation strategy.

**Impact:** Without proxies, MTProto sessions from a shared IP will be banned by Telegram within days. With proxies, cost could be $5-20/IP/month ($5K+/month at scale).

**Missing:** Provider selection, cost model, rotation strategy, failure handling.

### HIGH — Must fix before MVP launch

#### H1: Chat__c Relationship Type Wrong

**What:** `Messenger_Message__c.Chat__c` is implemented as Lookup. v11 spec requires Master-Detail (cascade delete, roll-up summaries like Attachment_Count__c).

**Impact:** Messages won't cascade-delete when chat is deleted. Roll-up summary fields (Attachment_Count__c) won't work.

**Note:** Changing Lookup to Master-Detail requires all existing Message records to have a non-null Chat__c. Plan migration carefully.

#### H2: No Sharing Model

**What:** `Messenger_Chat__c` has no sharing rules, no OWD configuration, no Apex Managed Sharing. The v11 spec documents OWD Private + Apex Managed Sharing derived from `Channel_User_Access__c`.

**Impact:** Either everyone sees all chats (security violation) or no one sees any chats (broken UX).

#### H3: No Queueable/Async for Outbound Callouts

**What:** `MessengerController.sendMessage()` makes a synchronous HTTP callout to the Go middleware. If Go is slow (network, proxy, Telegram rate limit), the LWC UI freezes.

**Impact:** Poor UX. Potential timeout errors (120 sec Salesforce callout limit).

**Fix:** Use Queueable Apex for outbound message sending. Return immediately to LWC, let Queueable handle delivery.

#### H4: Platform Event Delivery Limit at Scale

**What:** 50,000 deliveries per 24 hours. Each event x each subscriber = 1 delivery. With 3 PE types, 10 active chats, 5 agents, the math gets tight fast.

**Impact:** At ~16K messages/day with 3 agents, you hit the limit. Enterprise accounts with active messaging will exceed this.

**Mitigation already designed:** Centrifugo handles live UI (ADR-4). But PE are still used for durable persistence (triggers). Need to ensure PE triggers are efficient and subscribers are minimal.

#### H5: Pub/Sub API vs Platform Events — Unclear Routing

**What:** ADR-7 says "Hybrid ingestion: Platform Events + Bulk API + REST." ADR-8 says "Pub/Sub API (gRPC) for Go <-> Salesforce event streaming." Both address Go-to-SF data transfer but their roles overlap.

**Impact:** Developer confusion. Which API do you use for inbound messages? For delivery status? For session status?

**Needed:** Clear routing table: "Use PE for X, Pub/Sub for Y, REST for Z, Bulk API for W."

---

## 4. Architectural Concerns (Non-Blocking but Important)

### A1: BC Wallet Style Guide Copy-Paste

`style_guide_cursor_rules.md` is from a "BC Wallet" blockchain project, written in Russian. References TON, gRPC everywhere, `squad/v2` (not `oklog/run`), and 24MB message limits. None of this applies to MessageForge.

**Action:** Replace with MessageForge-specific Go style guide. The go-backend agent already enforces the correct patterns (oklog/run, local interfaces, slog).

### A2: oklog/run vs squad/v2 Confusion

The style guide says `squad/v2`. The Go rewrite design and actual code use `oklog/run`. These are different lifecycle management libraries.

**Resolution:** oklog/run is correct for MessageForge (already in go.mod, used in code). Remove squad references.

### A3: No Monitoring / Observability Strategy

No mention of: Prometheus metrics, structured log aggregation, alerting rules, SLOs, dashboards, distributed tracing.

**Impact:** In production, you're flying blind. Can't detect degradation, can't debug issues, can't prove SLA compliance.

### A4: No Cost Model

No pricing analysis for: Hetzner VPS, residential proxies, Salesforce Enterprise licenses, domain/SSL certificates. (**Note:** ADR-20/ADR-21 eliminated R2 and Centrifugo costs.)

**Impact:** Startup can't project burn rate, can't price the product, can't determine viable customer tier.

### A5: No Deployment Guide

No documentation for: how to deploy Go to Hetzner, how to configure TLS termination, how to run database migrations. (**Note:** ADR-20 and ADR-21 eliminated Centrifugo and Cloudflare R2/Workers, simplifying deployment requirements.)

### A6: JWT TTL Discrepancy

> **Note (ADR-21):** Centrifugo eliminated. The Centrifugo JWT TTL concern is no longer relevant. Only Salesforce OAuth token TTL (90 min) remains.

ADR-13 says Centrifugo JWTs are "15 min TTL." Security checklist mentions "90-min TTL." These are likely different systems (Centrifugo JWT vs Salesforce OAuth token), but the document doesn't clarify.

### A7: No Integration Testing Strategy

Backend has good unit tests (table-driven, mocks). No integration tests against real PostgreSQL, real Salesforce sandbox, or real Telegram API. No load testing plan.

### A8: Centrifugo Scaling Not Documented

> **SUPERSEDED (ADR-21):** Centrifugo eliminated. Real-time handled by Platform Events (empApi). This concern no longer applies.

Single Centrifugo instance. No HA, no clustering, no Redis backend for multi-node. What happens at 1000 concurrent WebSocket connections?

### A9: Media Cleanup / Orphan Detection Missing

> **Note (ADR-20):** R2 eliminated. Media now stored in Salesforce ContentVersion. Orphan risk shifts to ContentVersion records.

~~`media_files` table tracks R2 objects. No TTL policy, no orphan detection, no cleanup job. R2 storage will grow unbounded.~~ ContentVersion orphan detection and cleanup needed for Salesforce file storage management.

### A10: Missing ADR for Session Durability

UNLOGGED tables are a critical performance-vs-durability trade-off. This deserves a formal ADR (not just a comment in postgres-schema.md).

---

## 5. What's Excellent (Keep These)

1. **ADR discipline** — 17 decisions with clear context, alternatives, consequences. This is rare and valuable.

2. **Adapter pattern** — `MessengerAdapter` interface with `Platform()`, `Connect()`, `InboundMessages()`, `SendMessage()`, `DownloadMedia()` — clean, extensible, platform-agnostic.

3. **Security-first design** — HMAC-SHA256 with constant-time comparison, JWT Bearer Flow, presigned URLs with Content-Type binding, Protected CMTs for secrets.

4. **Performance awareness** — sync.Pool for batch buffers, SKIP LOCKED for queue processing, UNLOGGED tables for high-frequency writes, streaming JSON encoder.

5. **Claude Code configuration** — 4 focused agents, 9 domain skills, dev-review-loop for autonomous quality, question-resolver to avoid blocking on user. This is production-grade AI-assisted development.

6. **Salesforce for Go Developers guide** — Maps every SF concept to a Go/backend analogue. Invaluable for the primary developer's learning curve.

7. **Multi-platform foresight** — Generic `Messenger_*` naming, platform field everywhere, Config/App separation — adding WhatsApp will be additive, not restructuring.

---

## 6. Recommendations

### Immediate (This Week)

1. **Create ADR-18: Session Durability Strategy** — Formalize the UNLOGGED decision with recovery plan.
2. **Replace BC Wallet style guide** with MessageForge Go conventions.
3. **Create routing table** for Pub/Sub API vs Platform Events vs REST vs Bulk API.

### Before MVP (Next 4-6 Weeks)

4. **Implement v11 data model** — Create all 21 objects + 2 Protected CMTs. This is the foundation.
5. **Wire Go backend** — Connect nil dependencies (DB pool, media processor, ingester).
6. **Set up OWASP ZAP** — Run DAST scan against Go middleware early.
7. **Design sharing model** — OWD Private + Apex Managed Sharing.
8. **Protect Custom Metadata** — Migrate secrets to Protected CMT.

### Before Production

9. **Create deployment guide** — Hetzner, TLS, Go middleware. (Simplified by ADR-20/ADR-21: no R2, Workers, or Centrifugo.)
10. **Implement monitoring** — Prometheus metrics, log aggregation, alerting.
11. **Cost model** — Price every component, determine minimum viable customer tier.
12. **Residential proxy strategy** — Select provider, test, budget.

### Before AppExchange

13. **Complete DAST scan** with zero high-severity findings.
14. **SSL Labs scan** proving TLS 1.2+.
15. **Complete all security checklist items** (7 currently unchecked).
16. **Partner Community enrollment** and PADA negotiation.

---

## Appendices

- [Risk Register](./2026-03-29-risk-register.md) — All risks with severity, likelihood, impact, mitigation
- [Implementation Roadmap](./2026-03-29-implementation-roadmap.md) — Agent-executable phases with commands
