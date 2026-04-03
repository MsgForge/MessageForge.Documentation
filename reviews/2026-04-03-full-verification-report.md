# MessageForge Verification Report — 2026-04-03

## Executive Summary

MessageForge has a solid foundation (HMAC auth, encrypted credentials, Platform Events, trigger handlers, outbound pipeline) but two critical ADR violations block production readiness: **R2 media code is still active instead of ContentVersion (ADR-20)** and **Centrifugo code was never removed (ADR-21)**. Combined with missing FLS/CRUD checks on DML operations (AppExchange blocker) and 6/12 Go packages below 80% test coverage, the project scores **45/100 for deployment readiness**. Phase 1-3 of the MVP are substantially complete; Phase 4 (Media) is contradicted by stale R2 code; Phases 6-7 (Security Audit, Packaging) are not started.

---

## ADR Compliance

| ADR | Title | Status | Code Match | Issues |
|-----|-------|--------|------------|--------|
| ADR-1 | Support BOTH MTProto and Bot API | Accepted | PARTIAL | MTProto client exists but is PoC-grade, not fully wired into backend pipeline |
| ADR-2 | Remote PostgreSQL as Go-side data store | Accepted | YES | pgx/v5, migrations, pool -- all present |
| ADR-3 | ~~R2 + Workers for media~~ | Superseded (ADR-20) | N/A | -- |
| ADR-4 | ~~Centrifugo for real-time~~ | Superseded (ADR-21) | N/A | -- |
| ADR-5 | Single corporate API ID + proxy pool | Accepted | PARTIAL | Config accepts API ID/hash; proxy pool management not implemented |
| ADR-6 | HMAC-SHA256 for webhook auth | Accepted | YES | Go `middleware.go` + Apex `HMACValidator.cls` both correct, constant-time comparison |
| ADR-7 | Hybrid ingestion: PE + Bulk API + REST | Accepted | PARTIAL | Platform Events + REST implemented; Bulk API 2.0 not yet |
| ADR-8 | Pub/Sub API (gRPC) for streaming | Accepted | N/A | Explicitly future/non-MVP per ADR-19 |
| ADR-9 | Hetzner (Europe) as primary hosting | Accepted | YES | Deployment guide documents Hetzner CAX + Caddy |
| ADR-10 | Messenger-agnostic Go interface design | Accepted | YES | Platform-agnostic `Message` struct, adapter pattern |
| ADR-11 | 2GP Managed Package | Accepted | YES | `sfdx-project.json` with `tgint__` namespace |
| ADR-12 | Generic `Messenger_*` naming | Accepted | YES | All SF objects follow convention; `Messenger_Connection__c` fully migrated to `Messenger_Channel__c` |
| ADR-13 | ~~Centrifugo JWT auth~~ | Superseded (ADR-21) | N/A | -- |
| ADR-14 | ~~Direct-to-R2 media upload~~ | Superseded (ADR-20) | N/A | -- |
| ADR-15 | SRP v6a for MTProto 2FA | Accepted | YES | gotd/td crypto/srp library used in `client.go` |
| ADR-16 | Platform Events for delivery failure sync | Accepted | YES | `Message_Delivery_Status__e` + trigger + empApi subscription all working |
| ADR-17 | sync.Pool + streaming JSON for batch byte tracking | Accepted | **NO** | **Zero `sync.Pool` usage found in Go source.** Security checklist falsely claims it's implemented. |
| ADR-18 | UNLOGGED tables + periodic LOGGED backup | Accepted | YES | `001_initial.sql` UNLOGGED, `002_session_backup.sql` LOGGED backup, `session_backup.go` restore-on-startup |
| ADR-19 | Data Flow Routing Matrix | Accepted | PARTIAL | PE and REST paths implemented; Bulk API and Pub/Sub are documented as future |
| ADR-20 | Salesforce ContentVersion for all media | Accepted | **NO -- VIOLATION** | **R2 code is still active** (`r2.go`, `inbound.go`, `outbound.go`, `workers/media-proxy/`, config fields, migration column). **No ContentVersion Go code exists.** See Dead Code section. |
| ADR-21 | Platform Events (empApi) for real-time UI | Accepted | **NO -- VIOLATION** | **Centrifugo code not removed.** Go `centrifugo.go`, Apex `CentrifugoTokenController.cls`, LWC `centrifugoClient/`, `messengerLiveChat/`, CMT fields, docker-compose service all still present. empApi partially working in `messengerChat.js`. |

---

## Schema Alignment

### PostgreSQL

| Source | Documented | Actual | Match? | Issue |
|--------|-----------|--------|--------|-------|
| `connections` table | `credentials JSONB NOT NULL` only | Has both `credentials` (nullable) + `encrypted_credentials BYTEA NOT NULL` (migration 003-004) | PARTIAL | `encrypted_credentials` column not documented |
| `mtproto_sessions` table | Plain `REFERENCES connections(id)` | `UNIQUE` on `connection_id` + `ON DELETE CASCADE` | PARTIAL | Constraints not documented |
| `mtproto_sessions_backup` | `LIKE mtproto_sessions INCLUDING ALL` | Columns defined explicitly with `UNIQUE` on `connection_id` | PARTIAL | Implementation differs from documented approach |
| `media_files` table | "Eliminated per ADR-20" | Still created in `001_initial.sql` with `r2_object_key` column | **MISMATCH** | No drop migration exists; table created on fresh deploys |
| `inbound_queue` | Documented | Present | YES | -- |
| `outbound_queue` | Documented | Present | YES | -- |
| `schema_migrations` | Not documented | Present | N/A | Internal bookkeeping |

### Salesforce Data Model

| Source | Documented | Actual | Match? | Issue |
|--------|-----------|--------|--------|-------|
| `Session_Status__e.Channel_SF_ID__c` | Documented as `Channel_SF_ID__c` | Actual field is `Connection_SF_ID__c` | **MISMATCH** | All Apex code uses `Connection_SF_ID__c` -- doc is wrong |
| `Messenger_Event__c` | Not documented | 6-field object exists | **MISMATCH** | Undocumented object |
| `Messenger_Settings__mdt` | Not documented | CMT with `Go_Server_URL__c`, `Centrifugo_URL__c`, `Media_CDN_URL__c` | **MISMATCH** | Undocumented; contains stale Centrifugo field |
| `Middleware_Config__mdt` | Not documented | CMT with `HMAC_Secret__c`, `Go_Server_URL__c`, `Centrifugo_URL__c`, `Centrifugo_HMAC_Secret__c`, `Media_CDN_URL__c` | **MISMATCH** | Undocumented; contains stale Centrifugo fields |
| `Inbound_Message__e` fields | Partial field list | Missing 4 fields from docs: `Chat_SF_ID__c`, `Chat_External_ID__c`, `Media_MIME_Type__c`, `Protocol__c` | **MISMATCH** | Fields exist in code but not in sf-data-model.md |
| `Messenger_Message__c.Last_Error__c` | Not documented | Field exists alongside documented `Delivery_Error__c` | **MISMATCH** | Undocumented field |
| Go test `client_test.go:88` | -- | References `External_ID__c` on `Inbound_Message__e` | **MISMATCH** | No such field exists; likely should be `Chat_External_ID__c` |
| Go namespace in `ingestion.go` | Should use `tgint__Inbound_Message__e` | Uses `Inbound_Message__e` (no namespace) | **MISMATCH** | `delivery_status.go` correctly uses `tgint__` prefix, but ingester does not |

---

## Security Findings

| # | Severity | Area | Finding | File:Line | Recommendation |
|---|----------|------|---------|-----------|----------------|
| 1 | **CRITICAL** | Apex FLS | `ChannelSetupController.createChannel` and `createChannelWithConfig` perform `insert` without CRUD/FLS checks. Dynamic `createPlatformConfig` builds SObject from user-supplied `credentialsJson` without FLS validation. | `ChannelSetupController.cls:65,191,340-351` | Add `Security.stripInaccessible(AccessType.CREATABLE, records)` before every `insert`. Validate `isCreateable()` on dynamic fields. |
| 2 | **HIGH** | Apex FLS | `activateChannel` and `archiveChannel` query with `WITH SECURITY_ENFORCED` but `update` DML bypasses FLS. | `ChannelSetupController.cls:101,120` | Wrap updates with `Security.stripInaccessible(AccessType.UPDATABLE, ...)` |
| 3 | **HIGH** | Infra | `docker-compose.yml` has hardcoded Centrifugo secrets and `CENTRIFUGO_ALLOWED_ORIGINS: "*"`. No production guard. | `docker-compose.yml:26-30` | Remove Centrifugo service (ADR-21) or add `.env.example` pattern |
| 4 | **HIGH** | Go Config | Config still contains R2 + Centrifugo fields (`R2AccountID`, `R2AccessKeyID`, `R2SecretAccessKey`, `CentrifugoURL`, `CentrifugoAPIKey`, `CentrifugoSecret`). Dead config paths increase attack surface. | `config/config.go:31-40` | Remove dead config fields |
| 5 | **HIGH** | DB Migration | Migration 004 makes `encrypted_credentials NOT NULL` but does not drop legacy plaintext `credentials` JSONB column. | `004_use_encrypted_credentials.sql:8-16` | Create migration 005 to drop `credentials` column |
| 6 | **MEDIUM** | Apex HMAC | `HMACValidator.constantTimeEquals` leaks string length before constant-time loop (`a.length() != b.length()`). | `HMACValidator.cls:46` | Low practical risk (SHA-256 hex always 64 chars), but consider padding |
| 7 | **MEDIUM** | Go | Security checklist claims `sync.Pool buffers zeroed after use` -- not implemented anywhere. False assurance. | N/A (missing) | Implement or remove claim from checklist |
| 8 | **MEDIUM** | Apex/LWC | Dead Centrifugo code (`CentrifugoTokenController`, `centrifugoClient` LWC, `messengerLiveChat` LWC) adds unnecessary attack surface. | Multiple files | Remove per ADR-21 |
| 9 | **MEDIUM** | Go TLS | `JWTAuth` HTTP client lacks explicit TLS config. Go defaults are safe (TLS 1.2+) but not auditable. | `auth.go:178` | Add explicit `tls.Config{MinVersion: tls.VersionTLS12}` |
| 10 | **MEDIUM** | Go Rate Limit | No HTTP-level rate limiting on Go API endpoints (`/api/outbound`, webhooks). MTProto rate limiter only covers Telegram API. | `server/server.go` | Add HTTP middleware rate limiting |
| 11 | **LOW** | Apex | 5 classes use `without sharing` -- all have documented justifications (trigger context, sharing recalculation, compromise response). | Various `.cls` | No action needed; maintain documentation for AppExchange review |
| 12 | **LOW** | DB Schema | `media_files` table still has `r2_object_key` column from pre-ADR-20. | `001_initial.sql:63` | Drop column or repurpose for ContentVersion reference |

### Security PASS Items

- HMAC-SHA256 both sides with constant-time comparison
- OAuth JWT Bearer Flow (RS256, 5-min expiry)
- AES-256-GCM credential encryption (`crypto.go`)
- All SQL parameterized (`$1, $2` via pgx)
- No hardcoded secrets (env vars + Protected CMT)
- Input validation on API handlers
- MTProto sessions in UNLOGGED tables, never sent to SF
- LWC: no innerHTML/eval/localStorage, HTTPS-only media URLs
- 9/14 non-test Apex classes use `with sharing`
- All SOQL uses bind variables, no dynamic SOQL concatenation
- Error messages don't leak sensitive data

---

## MVP Progress

| Phase | Task | Status | Evidence |
|-------|------|--------|----------|
| **1 Foundation** | 1.1 Go Module & Dependencies | DONE | `go.mod`, `go.sum` |
| | 1.2 Configuration with Validation | DONE | `config/config.go` + `config_test.go` |
| | 1.3 PostgreSQL Connection Pool | DONE | `database/pool.go` + `pool_test.go` |
| | 1.4 Database Schema Migration | DONE | `database/migrations.go` + 4 migration files |
| | 1.5 HTTP Server Skeleton | DONE | `server/server.go`, `middleware.go`, `handlers.go` |
| | 1.6 Bot API Webhook Receiver | DONE | `telegram/botapi/handlers.go` + tests |
| **2 Core MTProto** | 2.1 MTProto Client Wrapper | DONE | `telegram/mtproto/client.go` |
| | 2.2 Auth Flow State Machine | PARTIAL | Auth methods in `client.go` but no dedicated `auth.go` or `auth_test.go` |
| | 2.3 Session Persistence | DONE | `mtproto/session_store.go` |
| | 2.4 Channel Message Parser | DONE | `mtproto/messages.go` + tests |
| | 2.5 Rate Limiting | DONE | `mtproto/ratelimit.go` + tests |
| | 2.6 Telegram Adapter | DONE | `telegram/adapter.go` + tests |
| **3 SF Integration** | 3.1 Custom Objects | DONE | All core objects + config objects created |
| | 3.2 Platform Events | DONE | 3 Platform Events with fields |
| | 3.3 Protected Custom Metadata | DONE | 4 Protected CMTs |
| | 3.4 Apex REST + HMAC | DONE | `MessengerInboundAPI.cls` + `HMACValidator.cls` |
| | 3.5 Platform Event Triggers | DONE | 3 triggers + handlers + tests |
| | 3.6 OAuth JWT Bearer (Go) | DONE | `salesforce/auth.go` + tests |
| | 3.7 Data Ingestion Pipeline | DONE | `salesforce/ingestion.go` + tests |
| | 3.8 Basic LWC Chat | DONE | `messengerChat/` with empApi |
| **4 Media Pipeline** | 4.1 ContentVersion REST Client | **NOT STARTED** | No `contentversion.go`. Media code still uses R2. |
| | 4.2 ContentDocumentLink Trigger | **NOT STARTED** | No trigger exists |
| | 4.3 Inbound Media (Telegram -> CV) | **CONTRADICTED** | `media/inbound.go` uploads to R2, not ContentVersion |
| | 4.4 Outbound Media (CV -> Telegram) | **CONTRADICTED** | `media/outbound.go` uses R2 presigned URLs |
| | 4.5 LWC Media Rendering | PARTIAL | `messengerMediaViewer/` LWC exists |
| **5 Real-time Chat** | 5.1 PE Publisher (Go) | PARTIAL | Publishes `Inbound_Message__e` + `Message_Delivery_Status__e`; no `Session_Status__e` publishing |
| | 5.2 empApi Subscription | DONE | `messengerChat.js` subscribes to events |
| | 5.3 empApi Error Handling | DONE | Auto-resubscription with 5s timer |
| | 5.4 Wire It All Together | PARTIAL | `pipeline.go` still references `realtimePublisher` (Centrifugo) |
| **6 Polish** | 6.1 Outbound Message Sending | DONE | Go handler + worker + SF service |
| | 6.2 Delivery Status Sync | DONE | Go `delivery_status.go` + SF trigger |
| | 6.3 Bot API Throttling | DONE | `botapi/throttle.go` + tests |
| | 6.4 Security Audit | NOT STARTED | No evidence |
| | 6.5 Permission Sets | PARTIAL | 3 permission sets (Admin, Agent, Viewer) vs plan's 2 (Admin, User) |
| **7 Packaging** | 7.1-7.3 | NOT STARTED | No package version, coverage report, or AppExchange prep |

---

## API Contract Mismatches

| Endpoint | Go Expects/Sends | SF Sends/Expects | Match? | Details |
|----------|-----------------|------------------|--------|---------|
| POST /api/outbound | `chatExternalId`, `text`, `messageType`, `mediaUrl`, `protocol`, `sfMessageExternalId` | `MessengerOutboundService.cls` sends identical fields | YES | -- |
| `Inbound_Message__e` | Go publishes 8 fields: `Text__c`, `Chat_External_ID__c`, `Sender_Name__c`, `Sender_External_ID__c`, `Message_Type__c`, `Media_URL__c`, `Sent_At__c`, `Protocol__c` | PE has 10 fields (extra: `Chat_SF_ID__c`, `Media_MIME_Type__c`) | PARTIAL | Go underutilizes schema; `Chat_SF_ID__c` path in trigger is dead for Go-originated events |
| `Message_Delivery_Status__e` | Go publishes: `Message_External_ID__c`, `Status__c`, `Error_Code__c`, `Error_Detail__c`, `Timestamp__c` | SF trigger reads same fields; maps DELIVERED/FAILED/RETRYING | YES | -- |
| HMAC config | Go: env var `WEBHOOK_SECRET`, `hmac.New(sha256.New)`, hex, `X-Signature` | SF: `Encryption_Key__mdt` where `DeveloperName='HMAC_Secret'`, `Crypto.generateMac('HmacSHA256')`, hex, `X-Signature` | YES | Same algorithm, encoding, header; different storage (env vs CMT) |
| Dual ingestion paths | Go publishes PE directly via Salesforce REST API | `MessengerInboundAPI.cls` accepts batch JSON and publishes PE itself | **UNCLEAR** | Two independent ingestion paths exist; unclear which is canonical |
| Namespace prefix | `ingestion.go` uses `Inbound_Message__e` (no prefix) | `delivery_status.go` uses `tgint__Message_Delivery_Status__e` | **MISMATCH** | Ingester should use `tgint__` prefix for managed package |

---

## Dead Code

| File | Type | Reason | ADR Reference |
|------|------|--------|---------------|
| `internal/media/r2.go` | Entire file (R2Client) | R2 eliminated | ADR-20 |
| `internal/media/r2_test.go` | Entire test file | R2 eliminated | ADR-20 |
| `internal/media/r2_mock_test.go` | Entire test file | R2 eliminated | ADR-20 |
| `internal/media/inbound.go` | Stale R2 references in comments | R2 eliminated | ADR-20 |
| `internal/media/outbound.go` | Stale R2 references | R2 eliminated | ADR-20 |
| `internal/common/models/models.go:32` | `MediaURL string // R2 URL` | R2 eliminated | ADR-20 |
| `internal/database/migrations/001_initial.sql:62` | `r2_object_key TEXT` column | R2 eliminated | ADR-20 |
| `internal/messenger/config/config.go:31-35` | R2 config fields | R2 eliminated | ADR-20 |
| `internal/messenger/config/config.go:38-40` | Centrifugo config fields | Centrifugo eliminated | ADR-21 |
| `internal/realtime/centrifugo.go` | Entire file (CentrifugoClient) | Centrifugo eliminated | ADR-21 |
| `internal/realtime/centrifugo_test.go` | Entire test file | Centrifugo eliminated | ADR-21 |
| `docker-compose.yml:19-37` | Centrifugo service definition | Centrifugo eliminated | ADR-21 |
| `workers/media-proxy/` | Cloudflare Worker (index.js + wrangler.toml) | R2 eliminated | ADR-20 |
| `draftdocs/CLAUDE.md` | Stale pre-ADR-20/21 doc | Extensive R2/Centrifugo refs | ADR-20, ADR-21 |
| SF: `CentrifugoTokenController.cls` + test | Entire Apex class | Centrifugo eliminated | ADR-21 |
| SF: `lwc/centrifugoClient/` | Entire LWC component | Centrifugo eliminated | ADR-21 |
| SF: `lwc/messengerLiveChat/` | LWC imports Centrifugo | Centrifugo eliminated | ADR-21 |
| SF: `Middleware_Config__mdt` Centrifugo fields | `Centrifugo_URL__c`, `Centrifugo_HMAC_Secret__c` | Centrifugo eliminated | ADR-21 |
| SF: `Messenger_Settings__mdt.Centrifugo_URL__c` | CMT field | Centrifugo eliminated | ADR-21 |
| SF: `Messenger_Attachment__c` description | "Files stored in R2" | R2 eliminated | ADR-20 |
| SF: `Middleware_Config__mdt.Media_CDN_URL__c` | "Cloudflare Worker media CDN proxy" | R2 eliminated | ADR-20 |
| SF: `.claude/skills/question-resolver.md:53` | "Media in R2 -- never store media in Salesforce" | **Directly contradicts ADR-20** | ADR-20 |

### Go Vet & TODOs

- `go vet ./...`: **Clean** -- no issues
- TODOs in Go: **None found**
- TODOs in Apex: `ChannelCompromiseTriggerHandler.cls:52` -- `// TODO: REST callout to Go middleware to disconnect session.`

---

## Test Coverage

| Package | Coverage | Critical? | Recommendation |
|---------|----------|-----------|----------------|
| `cmd/messenger` | 0.0% | Medium | Add smoke test for Init() |
| `internal/common/models` | No tests | Low | Add validation tests if models grow |
| `internal/database` | **49.0%** | HIGH | Needs significant coverage increase |
| `internal/media` | **70.4%** | Medium | Below 80%; much of this tests dead R2 code |
| `internal/messenger` | **63.0%** | HIGH | Below 80%; core app wiring needs more tests |
| `internal/messenger/config` | 100.0% | Low | Excellent |
| `internal/realtime` | 86.4% | **DEAD CODE** | Tests Centrifugo which should be removed |
| `internal/salesforce` | **79.9%** | HIGH | Just below 80% threshold |
| `internal/server` | 86.4% | HIGH | Good; includes HMAC tests |
| `internal/telegram` | 88.6% | HIGH | Good |
| `internal/telegram/botapi` | **78.8%** | HIGH | Below 80%; MVP protocol adapter |
| `internal/telegram/mtproto` | **77.4%** | Medium | Below 80%; lower priority if not MVP scope |

**Summary:** 6 of 12 packages below 80% threshold (excluding dead code). 32 test files total. `internal/database` at 49% is the worst offender. Dead R2/Centrifugo tests inflate overall numbers.

---

## Stale Documentation

| Document | Section | Issue | Recommendation |
|----------|---------|-------|----------------|
| `postgres-schema.md` | `connections` table | Missing `encrypted_credentials` column; still shows only `credentials JSONB NOT NULL` | Add `encrypted_credentials BYTEA NOT NULL` |
| `postgres-schema.md` | `media_files` table | Says "eliminated per ADR-20" but table still in migrations | Either drop table via migration or update doc |
| `postgres-schema.md` | `mtproto_sessions` | Missing UNIQUE and CASCADE constraints | Document actual constraints |
| `sf-data-model.md` | `Session_Status__e` | Documents `Channel_SF_ID__c`; actual field is `Connection_SF_ID__c` | Fix field name |
| `sf-data-model.md` | `Inbound_Message__e` | Missing 4 fields: `Chat_SF_ID__c`, `Chat_External_ID__c`, `Media_MIME_Type__c`, `Protocol__c` | Add missing fields |
| `sf-data-model.md` | Missing objects | `Messenger_Event__c`, `Messenger_Settings__mdt`, `Middleware_Config__mdt` not documented | Add to data model doc |
| `sf-data-model.md` | `Messenger_Message__c` | `Last_Error__c` field undocumented | Document or remove |
| `deployment-guide.md` | Env vars | Missing `TELEGRAM_BOT_TOKEN`, `ADMIN_HOST`, `ADMIN_PORT` | Add all 3; critical for MVP |
| `deployment-guide.md` | Docker | Does not mention Centrifugo service in docker-compose | Remove Centrifugo from compose (ADR-21) |
| `deployment-guide.md` | Firewall | Lists port 8080 only; missing 9090 (admin/metrics) | Add admin port |
| `security-checklist.md` | sync.Pool | Claims "sync.Pool buffers zeroed after use" -- not implemented | Remove false claim |
| `Messenger_Attachment__c` description | Object metadata | "Files stored in R2" | Update to "Files stored in ContentVersion" |
| `.claude/skills/question-resolver.md:53` | Rule 6 | "Media in R2 -- never store media in Salesforce" | **Delete rule** -- directly contradicts ADR-20 |
| `Backend/docs/wiki/data-model-and-relationships.md` | Throughout | References `sf_connection_id` and `Messenger_Connection__c` | Update to `sf_channel_id` / `Messenger_Channel__c` |
| `Backend/draftdocs/CLAUDE.md` | Entire file | Pre-ADR-20/21 content with extensive R2/Centrifugo references | Delete file |
| Architecture review (2026-03-29) | Implementation Status | Lists many objects as "NOT created" that now exist | Mark as substantially complete |

---

## Risk Register Status

| Risk ID | Original Finding | Current Status | Evidence |
|---------|-----------------|----------------|----------|
| R01 | Schema mismatch (Connection vs Channel) | RESOLVED | Zero `Messenger_Connection__c` references in `force-app/` |
| R02 | Unprotected secrets in CMT | RESOLVED | All 4 CMTs have `<visibility>Protected</visibility>` |
| R03 | No DAST scan | **OPEN** | No OWASP ZAP config or reports found. AppExchange blocker. |
| R04 | UNLOGGED sessions crash recovery | RESOLVED | `session_backup.go` + restore-on-startup |
| R05 | No HA / single point of failure | **OPEN** | No HA configuration or systemd units |
| R06 | Residential proxy not sourced | **OPEN** | No proxy provider integration |
| R07 | PE delivery exhaustion | **OPEN** | No PE monitoring or throttling code |
| R08 | Telegram ban risk | **OPEN** | Rate limiting exists but no behavioral randomization/jitter |
| R09 | No sharing model | PARTIALLY FIXED | `ChannelAccessService.cls` Apex Managed Sharing implemented; OWD not verifiable |
| R10 | Pub/Sub vs PE routing unclear | RESOLVED | ADR-19 routing table documented |
| R11 | Synchronous outbound callouts | **OPEN** | `MessengerController.cls:116` still synchronous; no Queueable pattern |
| R12 | BC Wallet style guide | RESOLVED | Correct MessageForge Go Style Guide |
| R13 | No monitoring/observability | RESOLVED | `server/metrics.go` Prometheus counters on separate admin port |
| R14 | No cost model | RESOLVED | Comprehensive `reviews/cost-model.md` |
| R15 | Media orphan growth | **OPEN** | No orphan detection code; concern shifted to ContentVersion |
| R16 | Centrifugo single instance | DEPRECATED | ADR-21 eliminated Centrifugo (but dead code remains) |
| R17 | JWT TTL discrepancy | RESOLVED | Only SF OAuth TTL remains; Centrifugo JWT eliminated |
| R21 | Queue worker duplicate processing | RESOLVED | Transaction wrapping in `queue_worker.go` |
| R22 | Ingester data loss on flush failure | RESOLVED | Copy-before-reset + lock-append-unlock rescue |
| R23 | Nil media downloader panic | RESOLVED | `pipeline.go` guards `p.media != nil` |
| R24 | Ingester payload field mismatch | RESOLVED | Correct field names match `Inbound_Message__e` |
| R25-R38 | Various code-level fixes | ALL RESOLVED | See fix register below |

### Fix Register: All 15 fixes verified -- no regressions

| Fix | What Was Fixed | Still Fixed? |
|-----|---------------|-------------|
| FIX-001 | Outbound JSON contract (SF->Go) | YES -- camelCase fields match |
| FIX-002 | SessionStatusTriggerHandler field name | YES -- uses `Connection_SF_ID__c` |
| FIX-003 | OutboundWorker never wired | YES -- `app.go:249` wires it |
| FIX-004 | Inbound queue dead path | YES -- `pipeline.go` enqueues on failure |
| FIX-005 | Encrypted credentials not wired | YES -- `connections.go` reads/writes encrypted |
| FIX-006 | EncryptionKey config no validation | YES -- `validate:"required"` |
| FIX-007 | archived/ breaks go vet | YES -- directory deleted |
| FIX-008 | Terminal error string matching | YES -- `TelegramError` struct with code map |
| FIX-009 | GlobalMetrics singleton | YES -- injected via constructor |
| FIX-010 | ContentType not allowlisted | YES -- `allowedContentTypes` map |
| FIX-011 | Chat_SF_ID__c naming | YES -- both ID fields coexist |
| FIX-012 | console.error() in LWC | YES -- zero occurrences |
| FIX-013 | Rollback uses cancelled context | YES -- `context.Background()` |
| FIX-014 | DML exception logging | YES -- detailed field/status/message |
| FIX-015 | Missing bulk test scenarios | YES -- multiple `testBulk*` methods |

---

## Action Items (Priority Order)

### CRITICAL (blocks AppExchange / production)

1. **[CRITICAL] Add FLS/CRUD checks to `ChannelSetupController.cls`** -- `insert` and `update` DML without `Security.stripInaccessible()`. AppExchange security review will reject this. (Security Finding #1, #2)

2. **[CRITICAL] Remove R2 code and implement ContentVersion (ADR-20)** -- `r2.go`, `r2_test.go`, `r2_mock_test.go`, `workers/media-proxy/`, R2 config fields, `r2_object_key` migration column, model comment. Then implement ContentVersion upload/download in Go. (ADR-20 violation)

3. **[CRITICAL] Remove Centrifugo code (ADR-21)** -- Go: `centrifugo.go`, `centrifugo_test.go`, docker-compose service, config fields. SF: `CentrifugoTokenController.cls` + test, `centrifugoClient/` LWC, `messengerLiveChat/` LWC, CMT Centrifugo fields. (ADR-21 violation)

4. **[CRITICAL] Run DAST scan** -- No OWASP ZAP or equivalent scan evidence. Hard AppExchange blocker. (Risk R03)

### HIGH

5. **[HIGH] Drop plaintext `credentials` column** -- Create migration 005 to drop `credentials JSONB` from `connections` table. Plaintext credentials may still exist at rest. (Security Finding #5)

6. **[HIGH] Fix namespace prefix in `ingestion.go`** -- Use `tgint__Inbound_Message__e` instead of `Inbound_Message__e`. (API Contract mismatch)

7. **[HIGH] Fix `client_test.go`** -- References nonexistent `External_ID__c` field on `Inbound_Message__e`. (Schema mismatch)

8. **[HIGH] Add HTTP-level rate limiting** -- No rate limiting on `/api/outbound` or webhook endpoints. (Security Finding #10)

9. **[HIGH] Raise test coverage** -- `internal/database` at 49%, `internal/messenger` at 63%. Both are critical paths. (Test coverage)

10. **[HIGH] Make outbound callouts async** -- `MessengerController.cls:116` makes synchronous HTTP callout. Use Queueable. (Risk R11)

### MEDIUM

11. **[MEDIUM] Update `sf-data-model.md`** -- Fix `Channel_SF_ID__c` -> `Connection_SF_ID__c`, add 4 missing `Inbound_Message__e` fields, add 3 undocumented objects, document `Last_Error__c`.

12. **[MEDIUM] Update `postgres-schema.md`** -- Add `encrypted_credentials` column, fix constraints, reconcile `media_files` status.

13. **[MEDIUM] Update `deployment-guide.md`** -- Add `TELEGRAM_BOT_TOKEN`, `ADMIN_HOST`, `ADMIN_PORT` env vars. Document admin/metrics port 9090.

14. **[MEDIUM] Implement `sync.Pool` or remove ADR-17 claim** -- Security checklist falsely claims implementation.

15. **[MEDIUM] Delete stale question-resolver rule** -- `.claude/skills/question-resolver.md:53` says "Media in R2" contradicting ADR-20.

16. **[MEDIUM] Add explicit TLS config to HTTP clients** -- `auth.go:178` lacks `tls.Config`. (Security Finding #9)

17. **[MEDIUM] Clarify dual ingestion paths** -- Go publishes PE directly; `MessengerInboundAPI.cls` also accepts batch JSON. Document which is canonical.

### LOW

18. **[LOW] Delete `draftdocs/CLAUDE.md`** -- Entirely stale pre-ADR-20/21 content.

19. **[LOW] Delete `MessageForge.Centrifugo/` directory** -- Placeholder with only CLAUDE.md.

20. **[LOW] Update architecture review (2026-03-29)** -- Many items marked "NOT created" now exist.

21. **[LOW] Add `Session_Status__e` publishing in Go** -- Platform Event exists but Go never publishes it.

22. **[LOW] Implement PE monitoring** -- No Platform Event delivery budget tracking. (Risk R07)

---

*Report generated by automated verification audit. All findings include file paths for traceability. No code was modified during this audit.*
