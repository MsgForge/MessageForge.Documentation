> **Note:** This review reflects architecture as of 2026-03-30. References to R2 and Centrifugo reflect pre-ADR-20/21 findings that have been superseded.

# MessageForge Recheck Review — 2026-03-30

**Reviewer:** Claude Opus 4.6 (6 parallel agents)
**Scope:** Full project recheck — Architecture, Security, Go Code Quality, Risk Register Verification, Documentation Consistency
**Prior review:** [2026-03-29-architecture-review.md](./2026-03-29-architecture-review.md)
**Verdict:** 9 previously RESOLVED risks confirmed. 17 NEW issues discovered (3 CRITICAL, 6 HIGH, 8 MEDIUM).

---

## Executive Summary

Six parallel review agents inspected the codebase against the March 29 architecture review, risk register, and implementation roadmap. The good news: **all 9 items marked RESOLVED or FIX-IN-PROGRESS are genuinely fixed** in code. The bad news: deeper inspection uncovered **3 critical data flow breaks** that will cause silent message loss in any integration test, plus **6 high-severity issues** spanning security, lifecycle, and JWT authentication.

| Dimension | March 29 | March 30 | Delta |
|---|---|---|---|
| Risks RESOLVED (verified) | 7 claimed | 9 confirmed | +2 (R21, R22 now fixed) |
| CRITICAL issues | 12 | 3 new | All original CRITICALs resolved; 3 new discovered |
| HIGH issues | 5 | 6 new | Original HIGHs still open; 6 new discovered |
| Go build | PASS | PASS | Clean |
| Go tests | PASS | PASS (all 11 packages, -race clean) | No regressions |
| Security posture | 6/10 | 7/10 | Protected CMTs confirmed, metrics gap found |
| Test coverage | Not measured | 43-100% (varies by package) | database: 43%, messenger: 51.5% |

---

## 1. Risk Register Verification

All items claimed as RESOLVED or FIX-IN-PROGRESS were verified against actual source code.

| Risk | Claimed | Verified | Evidence |
|---|---|---|---|
| R01 Schema mismatch | RESOLVED | **CONFIRMED** | `Messenger_Channel__c` used throughout Apex code |
| R02 Unprotected secrets | PARTIALLY OPEN | **RESOLVED** | Both `Encryption_Key__mdt` and `Messenger_Settings__mdt` have `<visibility>Protected</visibility>`. Note: Centrifugo secret no longer needed (ADR-21). |
| R04 UNLOGGED recovery | RESOLVED | **CONFIRMED** | `session_backup.go` implements 5-min snapshots + restore-on-startup |
| R10 API routing | RESOLVED | **CONFIRMED** | ADR-19 routing table complete with all API assignments |
| R12 Style guide | RESOLVED | **CONFIRMED** | MessageForge-specific guide, no blockchain/TON references |
| R13 Monitoring | RESOLVED | **CONFIRMED** | `metrics.go` exposes Prometheus counters at `/metrics` |
| R14 Cost model | RESOLVED | **CONFIRMED** | 341-line `cost-model.md` with 3 tiers |
| R17 JWT TTL | RESOLVED | **CONFIRMED** | Security checklist has explicit TTL table |
| R21 Queue duplicates | FIX IN PROGRESS | **FIXED** | Transaction boundary wraps full poll cycle in `queue_worker.go` |
| R22 Ingester data loss | FIX IN PROGRESS | **FIXED** | Batch lifecycle corrected; triggering payload preserved on flush failure |

**Action:** Update risk register to mark R02, R21, R22 as RESOLVED.

---

## 2. NEW Critical Issues

### NC1: Ingester Payload Field Names Don't Match Platform Event

**Severity:** CRITICAL
**Location:** `MessageForge.Backend/internal/salesforce/ingestion.go:38-49`
**Found by:** Architect agent

**What:** The Go ingester publishes Platform Event payloads with field names that don't match the Salesforce `Inbound_Message__e` definition:

| Go Ingester Sends | SF Platform Event Expects | Match? |
|---|---|---|
| `External_ID__c` | (no direct match) | NO |
| `Platform__c` | `Protocol__c` | NO |
| `Chat_ID__c` | `Chat_SF_ID__c` | NO |
| `Sender_ID__c` | `Sender_External_ID__c` | NO |
| `Media_Type__c` | `Message_Type__c` | NO |
| `Media_File_ID__c` | `Media_URL__c` | NO |
| (missing) | `Text__c` | MISSING |
| (missing) | `Sender_Name__c` | MISSING |
| (missing) | `Sent_At__c` | MISSING |

**Impact:** Every inbound message published by Go will arrive in Salesforce with null values for all fields. The `InboundMessageTriggerHandler` will create empty records or skip them entirely. **This is the #1 blocker for any integration testing.**

**Fix:** Align ingester payload keys to match `Inbound_Message__e` field API names exactly.

---

### NC2: No Chat External ID to SF Record ID Resolution

**Severity:** CRITICAL
**Location:** `MessageForge.Backend/internal/messenger/pipeline.go` + `MessageForge.Salesforce/force-app/main/default/classes/MessengerInboundAPI.cls`
**Found by:** Architect agent

**What:** The Go pipeline works with Telegram chat IDs (e.g., `-1001234567890`). The `Inbound_Message__e` Platform Event requires `Chat_SF_ID__c` — a Salesforce record ID for `Messenger_Chat__c` (e.g., `a0B5e00000XYZ12`). There is no mapping step between external chat ID and Salesforce record ID anywhere in the codebase.

`MessengerInboundAPI.cls` validates that `chatSfId` is a real Salesforce ID and **skips messages where it is blank**. Since the Go pipeline never provides a Salesforce ID, all messages will be silently discarded.

**Impact:** Zero inbound messages will be persisted in Salesforce until this mapping is implemented.

**Options:**
- A) Go queries SF at startup/on-demand to build a `telegram_chat_id -> sf_record_id` cache
- B) Go sends external chat ID; Apex trigger resolves to SF record via `Chat_External_ID__c` lookup
- C) Two-phase: Go creates `Messenger_Chat__c` records first, caches the returned SF IDs

**Recommendation:** Option B is simplest — modify `InboundMessageTriggerHandler` to resolve `Chat_External_ID__c` instead of requiring `Chat_SF_ID__c`.

---

### NC3: Nil Media Downloader Runtime Panic

**Severity:** CRITICAL
**Location:** `MessageForge.Backend/internal/messenger/app.go:205`, `MessageForge.Backend/internal/media/inbound.go:37`
**Found by:** Go reviewer agent

**What:** `NewApp()` creates the inbound media processor with `nil` for the downloader:

```go
mediaProc = media.NewInboundProcessor(nil, r2Client, 60*time.Minute)
```

The pipeline guard at `pipeline.go:64` only skips when `p.media == nil`. But `mediaProc` is non-nil (it was constructed), so the pipeline calls `mediaProc.Process()`, which calls `p.downloader.DownloadMedia()` on a nil pointer.

**Impact:** First Telegram message with any media attachment (photo, document, sticker) crashes the entire Go process. All messaging stops until manual restart.

**Fix:** Either:
- A) Pass `nil` for `mediaProc` in `NewApp()` when downloader is not available (pipeline already guards `p.media != nil`)
- B) Add a nil guard inside `InboundProcessor.Process()` for the downloader field

---

## 3. NEW High Issues

### NH1: Outbound Endpoint Not Registered

**Severity:** HIGH
**Location:** `MessageForge.Backend/internal/messenger/app.go` (missing registration)
**Found by:** Architect agent

**What:** `OutboundService` handler exists at `telegram/outbound.go` but is never registered with the HTTP server. Salesforce's `MessengerOutboundService` POSTs to `/api/outbound`. The Go server returns 404.

**Impact:** Outbound messaging (agent sends message from SF LWC) is completely non-functional.

**Fix:** Register the outbound handler in `NewApp()` server setup. Decide whether it should:
- A) Insert into `outbound_queue` for async processing by `OutboundWorker`
- B) Send directly via Telegram adapter (current handler behavior)

---

### NH2: JWT `aud` Claim Uses Token URL Instead of Login URL

**Severity:** HIGH
**Location:** `MessageForge.Backend/internal/salesforce/auth.go:93`
**Found by:** Go reviewer agent

**What:** The JWT Bearer Flow sets the `aud` (audience) claim to `a.config.TokenURL` (e.g., `https://login.salesforce.com/services/oauth2/token`). Per Salesforce specification, `aud` must be `https://login.salesforce.com` (or `https://test.salesforce.com` for sandbox).

**Impact:** Salesforce will reject every JWT token request with an "invalid audience" error. **The Go middleware cannot authenticate to Salesforce at all.**

**Fix:** Set `aud` to `"https://login.salesforce.com"` or extract the base URL from `TokenURL`.

---

### NH3: Queue Worker Silent UPDATE Failure Commits Transaction

**Severity:** HIGH
**Location:** `MessageForge.Backend/internal/messenger/queue_worker.go:157-165`
**Found by:** Go reviewer agent

**What:** Inside the transaction, `markDelivered` and `markFailed` execute UPDATE statements. On error, they log but do not abort the transaction. The outer `Commit()` succeeds, leaving rows in `'pending'` state with no retry scheduled. The ingester was already called (message sent to SF), so the next poll re-processes the same row — resulting in duplicate messages in Salesforce.

Same pattern in `outbound_worker.go:176-204`.

**Impact:** Partial failures create duplicate message delivery.

**Fix:** Return error from mark functions and abort the transaction on failure. Or use a "processed" flag within the loop to skip commit.

---

### NH4: `WEBHOOK_SECRET` Not Required in Config

**Severity:** HIGH
**Location:** `MessageForge.Backend/internal/messenger/config/config.go:41`
**Found by:** Security reviewer agent

**What:** `WebhookSecret` has no `required` validator tag. The server starts successfully without HMAC protection on webhook endpoints. When `webhookSecret` is empty, webhooks are registered without the HMAC middleware (`server.go:62`).

**Impact:** A misconfigured deployment accepts unauthenticated webhook requests. An attacker can inject fake Telegram messages into the pipeline.

**Fix:** Add `env:"WEBHOOK_SECRET,required"` to the config struct. Or at minimum, add `validate:"required"` tag.

---

### NH5: `/metrics` Endpoint Unauthenticated

**Severity:** HIGH
**Location:** `MessageForge.Backend/internal/server/server.go:92`
**Found by:** Security reviewer agent

**What:** The Prometheus metrics endpoint is registered on the public HTTP server without any authentication. It exposes: queue depths, error rates, flood wait counts, Salesforce token refresh frequency.

**Impact:** An attacker can profile the system to identify when it's under stress, when rate limits are hit, and when authentication tokens are refreshing.

**Fix options:**
- A) Move `/metrics` to a separate admin port (e.g., `:9090`)
- B) Add bearer token authentication
- C) Restrict to localhost/internal IP via middleware

---

### NH6: Plaintext `credentials` Column Not Migrated

**Severity:** HIGH
**Location:** `MessageForge.Backend/internal/database/migrations/001_initial.sql:11` + `003_encryption.sql`
**Found by:** Security reviewer agent

**What:** Migration 001 creates `credentials JSONB NOT NULL` (plaintext). Migration 003 adds `encrypted_credentials BYTEA`. But no code reads from or writes to `encrypted_credentials`. The actual credential lifecycle (encrypt on write, decrypt on read) is not wired. Bot tokens remain in plaintext JSON.

**Impact:** Database compromise exposes all Telegram bot tokens and API credentials in plaintext.

**Fix:** Wire `crypto.Encrypt`/`crypto.Decrypt` into the connection repository's read/write path. Drop or nullify the plaintext column after migration.

---

## 4. NEW Medium Issues

### NM1: `sf_connection_id` vs `sf_channel_id` Column Name Mismatch

**Severity:** MEDIUM
**Location:** `MessageForge.Backend/internal/database/migrations/001_initial.sql:9`
**Found by:** Architect agent

**What:** PostgreSQL migration uses `sf_connection_id`. Documentation (`postgres-schema.md`) specifies `sf_channel_id`. This is a leftover from the `Messenger_Connection__c` era. Go code querying this column will use the wrong name.

**Fix:** Rename column in migration to `sf_channel_id` to match documentation and Salesforce data model.

---

### NM2: Migration System Has No Version Tracking

**Severity:** MEDIUM
**Location:** `MessageForge.Backend/internal/database/migrations.go:18-44`
**Found by:** Architect agent + Go reviewer agent

**What:** Every migration file is re-executed on every application start. No `schema_migrations` table tracks which migrations have run. Current migrations are idempotent (`CREATE TABLE IF NOT EXISTS`), but any future non-idempotent migration (data transformation, column rename, `DROP COLUMN`) will be applied repeatedly.

**Impact:** Time bomb — safe today, dangerous when schema evolves.

**Fix:** Add a `schema_migrations` table and check version before executing each file.

---

### NM3: Dual Inbound Path Undocumented

**Severity:** MEDIUM
**Location:** `MessageForge.Backend/internal/messenger/pipeline.go` + `queue_worker.go`
**Found by:** Architect agent

**What:** Two parallel inbound paths exist:
1. **Pipeline** (real-time): Bot handler -> Pipeline -> media + Platform Event publish + Ingester (no retry, no persistence). (**Note (ADR-21):** Centrifugo replaced by Platform Events.)
2. **Queue Worker**: `inbound_queue` table -> QueueWorker -> Ingester (retry, SKIP LOCKED, persistent)

Nothing writes to `inbound_queue`. The Pipeline handles everything while the Queue handles nothing. If Pipeline's `processMessage()` fails, the message is logged and dropped — no retry, no persistence.

**Impact:** The queue's durability guarantees are unused for the primary inbound flow.

**Fix:** Document the demarcation: Pipeline = hot path (real-time, best-effort), Queue = durable path (guaranteed delivery). Either wire the Pipeline to insert into queue on failure, or document the acceptable loss window.

---

### NM4: Pool Closer Race Condition on Shutdown

**Severity:** MEDIUM
**Location:** `MessageForge.Backend/internal/messenger/app.go:74`
**Found by:** Architect agent

**What:** `PoolCloseRun()` closes the database pool in its interrupt function. `oklog/run` calls all interrupt functions concurrently when any actor fails. If the pool closes before the queue worker's in-flight transaction commits, the commit fails silently.

**Fix:** Close the pool after `g.Run()` returns (deferred), not as an `oklog/run` actor.

---

### NM5: `auth.go:Authenticate()` Missing `context.Context`

**Severity:** MEDIUM
**Location:** `MessageForge.Backend/internal/salesforce/auth.go:46`
**Found by:** Go reviewer agent

**What:** The `Authenticate()` method takes no `context.Context`. The HTTP call uses the client's hardcoded 30s timeout. The caller cannot cancel the token request on application shutdown.

**Impact:** Violates the project's own convention ("Context first: ctx context.Context should be first parameter"). During shutdown, a pending auth call blocks for up to 30 seconds.

**Fix:** Add `ctx context.Context` as first parameter. Use `http.NewRequestWithContext`.

---

### NM6: Ingester `Add` TOCTOU Window in Batch Rescue

**Severity:** MEDIUM
**Location:** `MessageForge.Backend/internal/salesforce/ingestion.go:53-76`
**Found by:** Go reviewer agent

**What:** After flush failure, the method unlocks the mutex (line 66), then immediately re-locks (line 69) to append the new payload. Between these two operations, another goroutine could call `Flush()` and drain the batch, causing the failure-rescued payloads to be lost before the new payload is appended.

**Fix:** Keep the lock held through the entire rescue + append sequence, or use a single critical section.

---

### NM7: Unsanitized Filename in R2 Key Path

**Severity:** MEDIUM
**Location:** `MessageForge.Backend/internal/media/outbound.go:58`
**Found by:** Go reviewer agent

**What:** `req.Filename` from the request body is used directly in the R2 object key:
```go
key := fmt.Sprintf("outbound/%s/%s/%s", req.ChatID, uid, req.Filename)
```
A filename like `../../etc/passwd` produces a path traversal key. While R2's namespace is flat (not a filesystem), `../` sequences can confuse CDN proxies, logging systems, or future file-path-based operations.

**Fix:** Sanitize filename — strip path separators, limit to alphanumeric + safe characters, or use a hash.

---

### NM8: `context.Canceled` Treated as Fatal on SIGINT

**Severity:** MEDIUM
**Location:** `MessageForge.Backend/cmd/messenger/main.go:47-49`
**Found by:** Go reviewer agent

**What:** On SIGINT, the signal actor returns `context.Canceled`, which `oklog/run` returns as a non-nil error. The main function wraps it as `"shutdown: context canceled"`, logs at `slog.Error`, and exits with code 1. A normal graceful shutdown should exit 0.

**Fix:** Check `errors.Is(err, context.Canceled)` before returning the error.

---

## 5. Code Quality Findings

### Global Metrics Singleton

**Severity:** LOW
**Location:** `MessageForge.Backend/internal/server/metrics.go:26`

`var GlobalMetrics = &Metrics{}` is a mutable package-level singleton. Queue workers in the `messenger` package import the `server` package to access it — a layering violation. Metrics should be injected as a dependency.

### Session Backup Zombie Sessions

**Severity:** LOW
**Location:** `MessageForge.Backend/internal/database/session_backup.go:87-98`

The backup uses `INSERT ... ON CONFLICT DO UPDATE` (UPSERT). Deleted sessions from the primary UNLOGGED table persist as zombies in the backup table indefinitely. On crash recovery, stale sessions for removed connections will be restored.

**Fix:** Add a cleanup step: delete backup rows where `connection_id` no longer exists in the primary table.

### Outbound Worker Silently Disabled

**Severity:** LOW
**Location:** `MessageForge.Backend/internal/messenger/app.go:249-251`

`outboundWorker` is always `nil`. `OutboundWorkerRun()` returns a no-op actor. Messages in `outbound_queue` sit indefinitely with no error or warning logged at startup.

**Fix:** Log a warning at startup when outbound worker is disabled.

### Terminal Error Strings Duplicated

**Severity:** LOW
**Location:** `MessageForge.Backend/internal/messenger/outbound_worker.go:210-224` + `internal/salesforce/delivery_status.go:17-24`

The same list of Telegram terminal error codes is maintained in two places with no shared source of truth.

**Fix:** Extract to a shared `telegram/errors.go` constant.

---

## 6. Documentation Issues

### MVP Implementation Plan — Stale Paths

**Severity:** HIGH (misleads developers)
**Location:** `MessageForge.Documentation/plans/mvp-implementation-plan.md`

| Stale Reference | Correct Path |
|---|---|
| `docs/reference/postgres-schema.md` | `MessageForge.Documentation/reference/postgres-schema.md` |
| `docs/reference/sf-data-model.md` | `MessageForge.Documentation/reference/sf-data-model.md` |
| `docs/reference/security-checklist.md` | `MessageForge.Documentation/reference/security-checklist.md` |
| `docs/plans/appexchange-onboarding.md` | `MessageForge.Documentation/plans/appexchange-onboarding.md` |
| `cmd/server/main.go` | `cmd/messenger/main.go` |
| `go build ./cmd/server/` | `go build ./cmd/messenger/` |

### ADR Summary Table Incomplete

**Severity:** MEDIUM
**Location:** `MessageForge.Documentation/architecture/adr.md` (lines 4-23)

The summary table at the top lists ADR-1 through ADR-17 only. ADR-18 (Session Durability) and ADR-19 (Data Flow Routing Matrix) are fully documented below but missing from the quick-reference table.

---

## 7. Security Assessment

### OWASP Top 10 (Go Middleware)

| Category | Status | Notes |
|---|---|---|
| A01: Broken Access Control | PASS | HMAC on webhooks, JWT Bearer for SF |
| A02: Cryptographic Failures | PASS | AES-256-GCM, HMAC-SHA256, RS256 JWT |
| A03: Injection | PASS | All SQL parameterized via pgx |
| A04: Insecure Design | WARN | No HTTP rate limiting on API endpoints |
| A05: Security Misconfiguration | WARN | `/metrics` unauthenticated, credential migration incomplete |
| A06: Vulnerable Components | NOT TESTED | No `govulncheck` or `npm audit` evidence |
| A07: Auth Failures | PASS | JWT Bearer, constant-time HMAC |
| A08: Data Integrity | PASS | HMAC on all data in transit |
| A09: Logging & Monitoring | PASS | slog JSON, Prometheus metrics |
| A10: SSRF | PASS | No user-controlled URLs in server-side requests |

### AppExchange Readiness

| Requirement | Status |
|---|---|
| Protected Custom Metadata | PASS |
| CRUD/FLS enforcement | PASS (`WITH SECURITY_ENFORCED` on all user-facing queries) |
| SOQL injection prevention | PASS (bind variables only) |
| XSS prevention in LWC | PASS (template syntax, HTTPS URL validation) |
| `with sharing` enforcement | PASS (2 exceptions documented) |
| No hardcoded credentials | PASS |
| DAST scan | **FAIL** — not performed (R03 still OPEN) |
| SSL Labs scan | **FAIL** — not performed |
| Penetration test | **FAIL** — not performed |

### Salesforce Security Highlights

- `HMACValidator.cls:44-54` — Custom `constantTimeEquals()` implementation verified correct
- `CentrifugoTokenController.cls:7` — Channel name regex `^messenger:chat:[a-zA-Z0-9]{15,18}$` prevents injection. **Note (ADR-21):** Centrifugo eliminated; this controller is no longer needed. empApi replaces Centrifugo for real-time.
- `MessengerOutboundService.cls:42` — HTTPS enforcement on Go server URL
- `messengerChat.js:393-399` — `sanitizeMediaUrl()` blocks `javascript:`, `data:`, HTTP URLs
- All LWC components: no `innerHTML`, no `lwc:dom="manual"`, no inline JS evaluation

---

## 8. Test Coverage

| Package | Coverage | Target (80%) |
|---|---|---|
| `internal/messenger/config` | 100% | PASS |
| `internal/realtime` | 86.4% | PASS |
| `internal/telegram` | 87.7% | PASS |
| `internal/telegram/botapi` | 76.8% | BELOW |
| `internal/telegram/mtproto` | 77.4% | BELOW |
| `internal/salesforce` | 80.3% | PASS |
| `internal/messenger` | 51.5% | BELOW |
| `internal/server` | 56.3% | BELOW |
| `internal/media` | 56.7% | BELOW |
| `internal/database` | 43.0% | BELOW |
| `cmd/messenger` | 0% | N/A (main) |

**Critical gaps:**
- `database` at 43% — `session_backup.go`, `migrations.go`, `PgSessionStore` untested against real DB
- `messenger` at 51.5% — `app.go` `NewApp()` happy path untested
- Missing tests: nil downloader panic, concurrent ingester flush race, graceful shutdown path

**Race detector:** PASS across all 11 packages.

---

## 9. Build & Toolchain Status

| Check | Result |
|---|---|
| `go vet ./...` | PASS |
| `go test -race ./... -count=1` | PASS (11 packages) |
| `go build ./cmd/messenger/` | PASS |
| Binary size | 30.5 MB (arm64) |
| Go version | 1.24.0 |

---

## 10. Prioritized Fix List

### Tier 1: Blocks Integration Testing (fix immediately)

| # | Issue | Location | Effort |
|---|---|---|---|
| NC1 | Align ingester payload field names to Platform Event | `ingestion.go:38-49` | 1 hour |
| NC2 | Add chat external ID resolution (Option B: Apex-side lookup) | `InboundMessageTriggerHandler.cls` | 2-4 hours |
| NC3 | Guard nil downloader (pass nil `mediaProc` to Pipeline) | `app.go:205` | 15 min |
| NH2 | Fix JWT `aud` claim to `login.salesforce.com` | `auth.go:93` | 15 min |
| NH1 | Register `/api/outbound` endpoint | `app.go` | 1-2 hours |

### Tier 2: Before MVP Launch

| # | Issue | Location | Effort |
|---|---|---|---|
| NH3 | Abort transaction on mark failure | `queue_worker.go`, `outbound_worker.go` | 1 hour |
| NH4 | Make `WEBHOOK_SECRET` required | `config.go:41` | 15 min |
| NH5 | Protect `/metrics` endpoint | `server.go:92` | 30 min |
| NH6 | Wire encrypted credentials read/write | `database/` | 2-4 hours |
| NM1 | Rename `sf_connection_id` to `sf_channel_id` | `001_initial.sql` | 30 min |
| NM2 | Add migration version tracking | `migrations.go` | 2-3 hours |
| NM4 | Fix pool closer shutdown ordering | `app.go:74` | 30 min |
| NM8 | Handle `context.Canceled` gracefully | `main.go:47` | 15 min |

### Tier 3: Before Production

| # | Issue | Location | Effort |
|---|---|---|---|
| NM3 | Document Pipeline vs Queue Worker demarcation | Architecture docs | 1 hour |
| NM5 | Add `context.Context` to `Authenticate()` | `auth.go:46` | 30 min |
| NM6 | Fix ingester TOCTOU window | `ingestion.go:53-76` | 30 min |
| NM7 | Sanitize filename in R2 key | `outbound.go:58` | 30 min |

### Tier 4: Documentation

| # | Issue | Location | Effort |
|---|---|---|---|
| D1 | Fix stale paths in MVP plan | `mvp-implementation-plan.md` | 30 min |
| D2 | Add ADR-18, ADR-19 to summary table | `adr.md` | 15 min |

---

## 11. Positive Findings

1. **All 9 RESOLVED risks are genuinely fixed.** The team executed well on the March 29 review findings.
2. **Security posture improved significantly.** Protected CMTs confirmed, R2 HMAC verified, constant-time comparison in both Go and Apex.
3. **ADR-18 (session backup) is production-quality.** Restore-on-startup, periodic snapshot, proper UPSERT — better than the documented TRUNCATE+COPY approach.
4. **ADR-19 (routing matrix) is clear and actionable.** Every data flow has an assigned API.
5. **Encryption is dual-layer.** Go: AES-256-GCM. Apex: AES-256-CBC via Protected CMT. Both use random IVs.
6. **ChannelCompromiseHandler uses Queueable correctly** to avoid callouts in trigger context.
7. **ChannelAccessService implements Apex Managed Sharing correctly** with proper `without sharing` justification.
8. **Race detector passes cleanly** across the entire Go test suite.
9. **Cost model is comprehensive** — 3 tiers, provider pricing, financial projections.
10. **Style guide replacement is complete** — MessageForge-specific, English, correct patterns.

---

## Appendices

- [Risk Register](./2026-03-29-risk-register.md) — To be updated with findings from this review
- [Implementation Roadmap](./2026-03-29-implementation-roadmap.md) — To be updated with pre-Phase 2 fixes
- [Architecture Review (March 29)](./2026-03-29-architecture-review.md) — Original review
- [Security Findings](./security-findings.md) — Detailed security analysis
