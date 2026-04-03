> **Note:** This risk register predates ADR-20 (ContentVersion, 2026-03-30) and ADR-21 (Platform Events, 2026-04-01). Risks referencing R2, Centrifugo, and Cloudflare Workers have been resolved by eliminating those components. Individual risks are marked DEPRECATED/SUPERSEDED inline where applicable.

# MessageForge Risk Register — 2026-03-29

**Last updated:** 2026-03-30
**Review cadence:** Update after each major phase completion

---

## Risk Scoring

- **Severity:** CRITICAL (blocks launch) | HIGH (degrades product) | MEDIUM (technical debt) | LOW (cosmetic)
- **Likelihood:** CERTAIN | LIKELY | POSSIBLE | UNLIKELY
- **Status:** OPEN | MITIGATED | ACCEPTED | RESOLVED

---

## CRITICAL Risks

### R01: Salesforce Schema Mismatch (Code vs Spec)

| Field | Value |
|---|---|
| Severity | CRITICAL |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Salesforce Dev |

**Description:** Actual code uses `Messenger_Connection__c` (6 fields). v11 spec defines `Messenger_Channel__c` (13+ fields) with 13 child objects. 13/21 documented objects don't exist.

**Impact:** Every Apex class, trigger, LWC, and Go middleware reference is wrong. Nothing works end-to-end until schema matches.

**Mitigation:**
1. Create all v11 objects in scratch org
2. Update all Apex/LWC references from `Messenger_Connection__c` to `Messenger_Channel__c`
3. Update Go middleware Salesforce client to use new API names
4. Verify with `sf apex run test --test-level RunLocalTests`

**Agent command:** `salesforce-dev` agent — "Implement full v11 data model: create all 21 custom objects, 2 Protected CMTs, update all Apex references"

---

### R02: Unprotected Secrets in Custom Metadata

| Field | Value |
|---|---|
| Severity | CRITICAL |
| Likelihood | CERTAIN |
| Status | RESOLVED (verified 2026-03-30) |
| Owner | Salesforce Dev |

**Description:** `Middleware_Config__mdt` was a regular (non-Protected) Custom Metadata Type.

**Resolution (verified 2026-03-30):** Both `Encryption_Key__mdt` and `Messenger_Settings__mdt` now have `<visibility>Protected</visibility>` in XML metadata. `HMACValidator.cls` and `CentrifugoTokenController.cls` read from `Encryption_Key__mdt` (Protected). `MessengerOutboundService.cls` reads HMAC from `Encryption_Key__mdt` and URL from `Messenger_Settings__mdt`. All Protected.

> **Note (ADR-21):** `CentrifugoTokenController.cls` and its associated Centrifugo HMAC secret are no longer needed. Centrifugo eliminated in favor of Platform Events (empApi).

---

### R03: No DAST/Penetration Testing

| Field | Value |
|---|---|
| Severity | CRITICAL |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Security |

**Description:** No DAST scan has been run on Go middleware. AppExchange requires zero high-severity findings from OWASP ZAP, Burp Suite, or Chimera.

**Impact:** Hard blocker for AppExchange publication. Unknown vulnerabilities in production.

**Mitigation:**
1. Deploy Go middleware to test environment (Docker on Hetzner)
2. Run OWASP ZAP automated scan against all endpoints
3. Fix any HIGH/CRITICAL findings
4. Document false positives
5. Generate PDF report for AppExchange submission

**Agent command:** `security-reviewer` agent — "Generate OWASP ZAP scan configuration for Go middleware endpoints"

---

### R04: UNLOGGED Sessions — No Crash Recovery

| Field | Value |
|---|---|
| Severity | CRITICAL |
| Likelihood | LIKELY |
| Status | OPEN |
| Owner | Go Backend |

**Description:** MTProto sessions stored in PostgreSQL UNLOGGED tables. Server crash or PG restart = all session data lost. Every user must re-authenticate.

**Impact:** Mass disconnection event. Poor UX. Support burden. At 100+ sessions, catastrophic.

**Mitigation options:**
- **Option A (recommended):** Keep UNLOGGED + implement periodic snapshot to LOGGED backup table (every 5 min). On crash, restore from backup. Users see brief reconnection, not full re-auth.
- **Option B:** Switch to LOGGED tables. ~2-3x slower writes but crash-safe. Acceptable if write frequency is manageable.
- **Option C:** Accept risk, document recovery SOP (notify users via SF Platform Event, auto-trigger re-auth).

**ADR needed:** ADR-18 — Session Durability Strategy

**Agent command:** `go-backend` agent — "Implement session backup mechanism: periodic snapshot from UNLOGGED mtproto_sessions to LOGGED mtproto_sessions_backup"

---

### R05: Single Point of Failure — No HA

| Field | Value |
|---|---|
| Severity | CRITICAL |
| Likelihood | LIKELY |
| Status | OPEN |
| Owner | Infrastructure |

**Description:** One Go middleware instance. No load balancing, failover, auto-restart, health monitoring.

**Impact:** Go crash = all messaging stops until manual restart.

**Mitigation:**
1. systemd service with auto-restart (Restart=always, RestartSec=5s)
2. Health check endpoint (`/health`) monitored externally (UptimeRobot, Healthchecks.io)
3. Document deployment with systemd unit file
4. Future: multi-instance with Redis for shared state

---

### R06: Residential Proxy — Not Sourced, Not Costed

| Field | Value |
|---|---|
| Severity | CRITICAL |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Infrastructure |

**Description:** ADR-5 mandates residential proxy pools for MTProto. Not sourced, not deployed, not budgeted. Many proxy providers explicitly block Telegram traffic.

**Impact:** Without proxies, MTProto from datacenter IP = Telegram ban within days. With wrong provider = wasted money + still banned.

**Mitigation:**
1. Research providers: Bright Data, Oxylabs, IPRoyal, SmartProxy — verify Telegram compatibility
2. Budget: $5-20/IP/month. For 50 sessions = $250-$1000/month
3. Test: validate 30-day session stability through proxy
4. Fallback: if proxies don't work, Bot API-only mode (degraded but functional)

---

## HIGH Risks

### R07: Platform Event Delivery Exhaustion

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | LIKELY |
| Status | OPEN — Partially mitigated by Centrifugo (ADR-4) |
| Owner | Architecture |

**Description:** 50,000 PE deliveries/24h. Each event x each subscriber = 1 delivery. With 3 PE types and multiple trigger subscribers, limit approaches fast.

**Impact:** At ~16K messages/day with 3+ subscribers, delivery stops. Triggers stop firing. Messages accumulate in Go queue without SF acknowledgment.

> **Note (ADR-21):** Centrifugo was eliminated. Real-time UI now uses Platform Events via empApi, which shares the same 50K delivery limit. This makes PE budget management more critical than before.

**Mitigation:**
- Minimize PE trigger subscribers (1 per PE type)
- Batch PE publishing (already implemented: 950KB threshold)
- Monitor `SELECT COUNT(*) FROM EventBusSubscriber WHERE EventType = 'tgint__Inbound_Message__e'`
- Future: Replace PE with Pub/Sub API (ADR-8) for higher throughput

---

### R08: Telegram Ban Risk

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | POSSIBLE |
| Status | OPEN |
| Owner | Go Backend |

**Description:** Telegram actively detects and bans automated accounts. Even with residential proxies, behavioral patterns (uniform message timing, bulk operations, 24/7 uptime) can trigger bans.

**Impact:** Account banned = channel goes offline. Customer loses access to their Telegram integration.

**Mitigation:**
1. Rate limiting already implemented (token bucket + FLOOD_WAIT)
2. Add behavioral randomization (jitter on message timing)
3. Respect all FLOOD_WAIT delays (already designed)
4. Monitor for AUTH_KEY_UNREGISTERED and USER_DEACTIVATED errors
5. Document ban recovery SOP for customers
6. Offer Bot API-only mode as fallback (no ban risk)

---

### R09: No Sharing Model

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Salesforce Dev |

**Description:** v11 spec documents OWD Private + Apex Managed Sharing derived from Channel_User_Access__c. Not implemented. Currently no sharing rules at all.

**Impact:** Either all users see all chats (data leak) or no users see any chats (broken UX).

**Mitigation:**
1. Create `Channel_User_Access__c` junction object
2. Set OWD to Private for Messenger_Chat__c
3. Implement `ChannelAccessService.cls` for Apex Managed Sharing
4. Create `ChannelAccessTrigger.trigger` for sharing recalculation
5. Test with multi-user scratch org

**Agent command:** `salesforce-dev` agent — "Implement Apex Managed Sharing for Messenger_Chat__c based on Channel_User_Access__c"

---

### R10: Pub/Sub API vs Platform Events — Unclear Routing

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Architecture |

**Description:** ADR-7 and ADR-8 both address Go<->SF data transfer. No clear routing table specifying which API to use for each data flow.

**Impact:** Developer confusion, inconsistent implementation, potential duplicate pathways.

**Proposed routing table:**

| Data Flow | Direction | API | Why |
|---|---|---|---|
| New inbound messages (real-time) | Go -> SF | Platform Events (Inbound_Message__e) | Low latency, trigger-driven record creation |
| Session status changes | Go -> SF | Platform Events (Session_Status__e) | Infrequent, critical for UI |
| Delivery status updates | Go -> SF | Platform Events (Message_Delivery_Status__e) | Real-time UI feedback |
| Outbound message requests | SF -> Go | REST API (callout from Queueable) | Request-response pattern |
| Bulk message archive | Go -> SF | Bulk API 2.0 | High volume, async acceptable |
| Channel config at startup | Go <- SF | REST API (query) | One-time read |
| Bidirectional streaming (future) | Go <-> SF | Pub/Sub API (gRPC) | When PE delivery limit is hit |

**ADR needed:** ADR-19 — Data Flow Routing Matrix

---

### R11: Synchronous Outbound Callouts

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | LIKELY |
| Status | OPEN |
| Owner | Salesforce Dev |

**Description:** `MessengerController.sendMessage()` makes synchronous HTTP callout to Go. If Go is slow, LWC UI hangs.

**Impact:** Poor UX. Timeout errors (120 sec limit). User clicks send, waits, gets error.

**Mitigation:** Use Queueable Apex. Return "pending" immediately to LWC. Let Queueable handle delivery + status update via Platform Event.

---

## MEDIUM Risks

### R12: BC Wallet Style Guide — Wrong Project

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Documentation |

**Description:** `style_guide_cursor_rules.md` is from a blockchain project, in Russian, references squad/v2, TON, gRPC defaults. Does not match MessageForge.

**Mitigation:** Replace with MessageForge Go style guide derived from go-backend agent rules + actual code patterns.

---

### R13: No Monitoring / Observability

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Infrastructure |

**Description:** No Prometheus metrics, no log aggregation, no alerting, no dashboards, no SLOs.

**Impact:** Can't detect degradation, debug production issues, or prove reliability to customers.

**Mitigation:**
1. Add Prometheus metrics to Go middleware (message count, latency, queue depth, error rate)
2. Structured logging already in place (slog JSON) — add log aggregation (Loki, CloudWatch)
3. Define SLOs: message delivery < 5s, availability > 99.5%
4. Set up alerting (PagerDuty/Slack) for: queue backup, Telegram FLOOD_WAIT, SF token expiry

---

### R14: No Cost Model

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Business |

**Description:** No pricing analysis for any infrastructure component.

**Estimated costs (monthly):**

| Component | Estimate | Notes |
|---|---|---|
| Hetzner VPS (ARM, 4GB) | $5-15 | Go middleware |
| Hetzner Managed PostgreSQL | $15-30 | Or self-hosted |
| Residential proxies (50 IPs) | $250-1000 | Biggest variable cost |
| Salesforce ContentVersion | $0 | Included in subscriber's org file storage (ADR-20) |
| Domain + SSL | $10-20/year | Let's Encrypt for free SSL |
| **Total (MVP)** | **$280-1070/month** | Proxies dominate |

---

### R15: Media Orphan Growth

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | LIKELY |
| Status | OPEN |
| Owner | Go Backend |

> **Note (ADR-20):** R2 eliminated. Media now stored in Salesforce ContentVersion. Orphan risk shifts to ContentVersion/ContentDocument records without valid parent links. Cleanup strategy must use Salesforce queries instead of PostgreSQL `media_files` table.

**Description:** ~~R2 objects tracked in `media_files` table. No TTL, no cleanup job, no orphan detection.~~ ContentVersion records linked to deleted messages or attachments may persist as orphans.

**Impact:** Unbounded Salesforce file storage growth, consuming subscriber's org allocation ($5/GB/month for excess).

**Mitigation:** Implement periodic cleanup: query ContentDocumentLink for orphaned links, delete ContentDocument records where all parent links are to deleted records.

---

### R16: Centrifugo Single Instance

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | POSSIBLE |
| Status | **DEPRECATED (ADR-21)** |
| Owner | Infrastructure |

> **DEPRECATED (ADR-21):** Centrifugo eliminated. Real-time handled by Platform Events (empApi). This risk is no longer applicable.

**Description:** Single Centrifugo node. No HA, no Redis backend for multi-node.

**Impact:** Centrifugo crash = live chat stops. Users see stale data until refresh.

**Mitigation:** For MVP, auto-restart via systemd is sufficient. For production, add Redis backend and second Centrifugo node.

---

### R17: JWT TTL Discrepancy

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | POSSIBLE |
| Status | OPEN |
| Owner | Documentation |

**Description:** ADR-13 says "15 min" for Centrifugo JWT. Security checklist says "90-min TTL." Likely different systems (Centrifugo vs OAuth) but document doesn't clarify.

> **Note (ADR-21):** Centrifugo eliminated. Centrifugo JWT TTL is no longer relevant. Only Salesforce OAuth token TTL (90 min) remains.

**Mitigation:** ~~Add explicit table: "Centrifugo JWT: 15 min TTL. Salesforce OAuth token: 90 min TTL. Presigned URLs: 15-60 min TTL."~~ Simplified: only Salesforce OAuth token TTL (90 min) needs to be documented.

---

## LOW Risks

### R18: Settings.json Proliferation

| Severity | LOW | 9 settings files across 6 sub-projects + root |
|---|---|---|

**Mitigation:** Consolidate or document inheritance model.

### R19: Duplicated Skills in Sub-Projects

| Severity | LOW | MessageForge.Salesforce replicates all root-level skills |
|---|---|---|

**Mitigation:** Remove duplicates, reference root-level skills.

### R20: No Admin Dashboard Design

| Severity | LOW | MessageForge.Admin is empty placeholder |
|---|---|---|

**Mitigation:** Deprioritize until post-MVP. Document scope in CLAUDE.md.

---

## NEW Risks (Identified 2026-03-29)

### R21: Queue Worker Duplicate Processing

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | LIKELY |
| Status | RESOLVED (verified 2026-03-30) |
| Owner | Go Backend |

**Description:** `FOR UPDATE SKIP LOCKED` was used without explicit transaction boundary.

**Resolution (verified 2026-03-30):** `queue_worker.go` now wraps the entire poll cycle in a single transaction: `Begin -> SELECT FOR UPDATE SKIP LOCKED -> process -> UPDATE -> Commit`. Lock held until commit. Verified in code review.

**Note:** A residual issue remains — `markDelivered`/`markFailed` UPDATE errors are logged but don't abort the transaction. See R25 (NEW).

---

### R22: Ingester Data Loss on Flush Failure

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | POSSIBLE |
| Status | RESOLVED (verified 2026-03-30) |
| Owner | Go Backend |

**Description:** `Ingester.Add()` pre-emptively reset the batch when full. If flush failed, batch was lost.

**Resolution (verified 2026-03-30):** Batch is now copied to local variable before reset. On flush failure, triggering payload is re-added to the fresh batch (lines 62-66 of `ingestion.go`). Verified in code review.

**Note:** A residual TOCTOU window exists between unlock-after-rescue and re-lock-for-append. See R31 (NEW).

---

### R23: Nil Media Downloader — Latent Panic Risk

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | LIKELY |
| Status | OPEN — Upgraded to CRITICAL (2026-03-30) |
| Owner | Go Backend |

**Description:** `app.go:205` passes `nil` for the downloader when constructing `InboundProcessor`. The pipeline guard at `pipeline.go:64` only skips when `p.media == nil`, but `mediaProc` is non-nil (constructed with nil downloader). First media message triggers nil pointer dereference.

**Code location:** `MessageForge.Backend/internal/messenger/app.go:205` + `internal/media/inbound.go:37`

**Impact:** Any Telegram message with media (photo, document, sticker) crashes the entire Go process. All messaging stops.

**Fix (immediate):** Pass `nil` for `mediaProc` when downloader is unavailable. Pipeline already guards `p.media != nil`.

**Timeline:** Fix immediately (15 min). Full media integration in Phase 2.

---

## NEW Risks (Identified 2026-03-30)

### R24: Ingester Payload Field Names Don't Match Platform Event

| Field | Value |
|---|---|
| Severity | CRITICAL |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Go Backend |

**Description:** Go ingester (`ingestion.go:38-49`) publishes Platform Event payloads with field names that don't match `Inbound_Message__e`: `Chat_ID__c` vs `Chat_SF_ID__c`, `Sender_ID__c` vs `Sender_External_ID__c`, `Media_Type__c` vs `Message_Type__c`, `Media_File_ID__c` (doesn't exist). Missing fields: `Text__c`, `Sender_Name__c`, `Sent_At__c`.

**Impact:** Every inbound message arrives in Salesforce with null values. `InboundMessageTriggerHandler` creates empty records or skips entirely. **Blocks all integration testing.**

**Fix:** Align ingester payload keys to match `Inbound_Message__e` field API names exactly.

**Effort:** 1 hour

---

### R25: No Chat External ID to SF Record ID Resolution

| Field | Value |
|---|---|
| Severity | CRITICAL |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Go Backend + Salesforce Dev |

**Description:** Go pipeline works with Telegram chat IDs (e.g., `-1001234567890`). `Inbound_Message__e` requires `Chat_SF_ID__c` — a Salesforce record ID. No mapping exists. `MessengerInboundAPI.cls` skips messages with blank `chatSfId`.

**Impact:** Zero inbound messages persisted in Salesforce.

**Options:**
- A) Go queries SF to build `telegram_chat_id -> sf_record_id` cache
- B) Modify `InboundMessageTriggerHandler` to resolve via `Chat_External_ID__c` lookup (recommended)
- C) Two-phase: Go creates `Messenger_Chat__c` first, caches returned SF IDs

**Effort:** 2-4 hours

---

### R26: JWT `aud` Claim Uses Token URL Instead of Login URL

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Go Backend |

**Description:** `auth.go:93` sets JWT `aud` to `a.config.TokenURL` (full token endpoint). Salesforce requires `aud` to be `https://login.salesforce.com` (or `test.salesforce.com` for sandbox).

**Impact:** Salesforce rejects every JWT token request. Go middleware cannot authenticate to SF at all.

**Fix:** Set `aud` to `"https://login.salesforce.com"` or extract base URL from `TokenURL`.

**Effort:** 15 minutes

---

### R27: Outbound Endpoint `/api/outbound` Not Registered

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Go Backend |

**Description:** `OutboundService` handler exists at `telegram/outbound.go` but is never registered with the HTTP server in `NewApp()`. Salesforce `MessengerOutboundService` POSTs to `/api/outbound` and gets 404.

**Impact:** Outbound messaging completely non-functional.

**Fix:** Register handler in `NewApp()`. Decide: insert into `outbound_queue` (async) or send directly (current handler behavior).

**Effort:** 1-2 hours

---

### R28: `WEBHOOK_SECRET` Not Required in Config

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | POSSIBLE |
| Status | OPEN |
| Owner | Go Backend |

**Description:** `config.go:41` `WebhookSecret` has no `required` validator tag. Server can start without HMAC protection. When empty, webhooks are registered without HMAC middleware (`server.go:62`).

**Impact:** Misconfigured deployment accepts unauthenticated webhook requests. Attacker can inject fake messages.

**Fix:** Add `env:"WEBHOOK_SECRET,required"` or `validate:"required"`.

**Effort:** 15 minutes

---

### R29: `/metrics` Endpoint Unauthenticated

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | POSSIBLE |
| Status | OPEN |
| Owner | Go Backend |

**Description:** Prometheus `/metrics` endpoint at `server.go:92` has no authentication. Exposes queue depths, error rates, flood wait counts, token refresh frequency.

**Impact:** Attacker can profile system stress patterns and timing.

**Fix:** Move to separate admin port, add bearer token, or restrict to localhost.

**Effort:** 30 minutes

---

### R30: Plaintext `credentials` Column Not Migrated to Encrypted

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | LIKELY |
| Status | OPEN |
| Owner | Go Backend |

**Description:** Migration 001 creates `credentials JSONB NOT NULL` (plaintext). Migration 003 adds `encrypted_credentials BYTEA`. No code reads/writes `encrypted_credentials`. AES-256-GCM implementation exists in `crypto.go` but is not wired into the connection repository.

**Impact:** Database compromise exposes all bot tokens and API credentials in plaintext.

**Fix:** Wire `crypto.Encrypt`/`crypto.Decrypt` into connection read/write path. Drop or nullify plaintext column.

**Effort:** 2-4 hours

---

### R31: Queue Worker Silent UPDATE Failure Commits Transaction

| Field | Value |
|---|---|
| Severity | HIGH |
| Likelihood | POSSIBLE |
| Status | OPEN |
| Owner | Go Backend |

**Description:** Inside the transaction in `queue_worker.go`, `markDelivered`/`markFailed` UPDATE errors are logged but don't abort the transaction. `Commit()` succeeds, leaving rows in `pending` state. Next poll re-processes the same row — duplicate message in Salesforce.

Same pattern in `outbound_worker.go:176-204`.

**Impact:** Partial failures create duplicate message delivery.

**Fix:** Return error from mark functions and abort transaction on failure.

**Effort:** 1 hour

---

### R32: `sf_connection_id` vs `sf_channel_id` Column Name Mismatch

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Go Backend |

**Description:** Migration `001_initial.sql:9` uses `sf_connection_id`. Documentation (`postgres-schema.md`) specifies `sf_channel_id`. Leftover from `Messenger_Connection__c` era.

**Fix:** Rename column in migration to `sf_channel_id`.

**Effort:** 30 minutes

---

### R33: Migration System Has No Version Tracking

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | LIKELY |
| Status | OPEN |
| Owner | Go Backend |

**Description:** Every migration file re-executes on every startup (`migrations.go:18-44`). No `schema_migrations` table. Current migrations are idempotent, but any future non-idempotent migration (data transform, column rename) will be applied repeatedly.

**Impact:** Time bomb — safe today, dangerous when schema evolves.

**Fix:** Add `schema_migrations` table and version check before executing each file.

**Effort:** 2-3 hours

---

### R34: Pool Closer Race Condition on Shutdown

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | POSSIBLE |
| Status | OPEN |
| Owner | Go Backend |

**Description:** `PoolCloseRun()` at `app.go:74` closes the database pool in its interrupt function. `oklog/run` calls all interrupt functions concurrently. Pool may close before queue worker's in-flight transaction commits.

**Fix:** Close pool after `g.Run()` returns (deferred), not as an `oklog/run` actor.

**Effort:** 30 minutes

---

### R35: Unsanitized Filename in R2 Key Path

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | POSSIBLE |
| Status | OPEN |
| Owner | Go Backend |

**Description:** `outbound.go:58` uses `req.Filename` directly in R2 object key. Path traversal sequences (`../../`) could confuse CDN proxies or logging.

**Fix:** Sanitize filename — strip path separators, limit to safe characters.

**Effort:** 30 minutes

---

### R36: `context.Canceled` Treated as Fatal on SIGINT

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | CERTAIN |
| Status | OPEN |
| Owner | Go Backend |

**Description:** `main.go:47-49` wraps `context.Canceled` as error, logs at `slog.Error`, exits code 1. Normal graceful shutdown should exit 0.

**Fix:** Check `errors.Is(err, context.Canceled)` before returning.

**Effort:** 15 minutes

---

### R37: Ingester TOCTOU Window in Batch Rescue

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | POSSIBLE |
| Status | OPEN |
| Owner | Go Backend |

**Description:** `ingestion.go:53-76` — between unlock-after-rescue (line 66) and re-lock-for-append (line 69), another goroutine could call `Flush()` and drain the rescued batch.

**Fix:** Keep lock held through entire rescue + append sequence.

**Effort:** 30 minutes

---

### R38: `Authenticate()` Missing `context.Context`

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Likelihood | LIKELY |
| Status | OPEN |
| Owner | Go Backend |

**Description:** `auth.go:46` `Authenticate()` takes no context. HTTP call uses hardcoded 30s timeout. Caller cannot cancel on shutdown.

**Fix:** Add `ctx context.Context` as first parameter.

**Effort:** 30 minutes

---

## Risk Summary Matrix

| ID | Risk | Severity | Likelihood | Status |
|---|---|---|---|---|
| R01 | Schema mismatch | CRITICAL | CERTAIN | RESOLVED (2026-03-29) |
| R02 | Unprotected secrets | CRITICAL | CERTAIN | RESOLVED (verified 2026-03-30) |
| R03 | No DAST scan | CRITICAL | CERTAIN | OPEN |
| R04 | UNLOGGED crash recovery | CRITICAL | LIKELY | RESOLVED (2026-03-29) |
| R05 | No HA | CRITICAL | LIKELY | OPEN |
| R06 | Proxy not sourced | CRITICAL | CERTAIN | OPEN |
| R07 | PE delivery exhaustion | HIGH | LIKELY | Partial |
| R08 | Telegram ban | HIGH | POSSIBLE | OPEN |
| R09 | No sharing model | HIGH | CERTAIN | OPEN |
| R10 | API routing unclear | HIGH | CERTAIN | RESOLVED (2026-03-29) |
| R11 | Sync callouts | HIGH | LIKELY | OPEN |
| R12 | Wrong style guide | MEDIUM | CERTAIN | RESOLVED (2026-03-29) |
| R13 | No monitoring | MEDIUM | CERTAIN | RESOLVED (2026-03-29) |
| R14 | No cost model | MEDIUM | CERTAIN | RESOLVED (2026-03-29) |
| R15 | Media orphans | MEDIUM | LIKELY | OPEN |
| R16 | Centrifugo single node | MEDIUM | POSSIBLE | DEPRECATED (ADR-21) |
| R17 | JWT TTL unclear | MEDIUM | POSSIBLE | RESOLVED (2026-03-29) |
| R18 | Settings proliferation | LOW | CERTAIN | OPEN |
| R19 | Duplicated skills | LOW | CERTAIN | OPEN |
| R20 | No admin design | LOW | CERTAIN | OPEN |
| R21 | Queue worker duplicates | HIGH | LIKELY | RESOLVED (verified 2026-03-30) |
| R22 | Ingester data loss | HIGH | POSSIBLE | RESOLVED (verified 2026-03-30) |
| R23 | Nil media downloader | **CRITICAL** | LIKELY | RESOLVED (2026-03-30) — defensive guard + pipeline nil check |
| R24 | Ingester payload field mismatch | **CRITICAL** | CERTAIN | RESOLVED (2026-03-30) — fields aligned to Inbound_Message__e |
| R25 | No chat ID to SF record ID mapping | **CRITICAL** | CERTAIN | RESOLVED (2026-03-30) — Apex-side lookup + auto-create |
| R26 | JWT `aud` claim wrong | HIGH | CERTAIN | RESOLVED (2026-03-30) — extracts base URL from TokenURL |
| R27 | Outbound endpoint not registered | HIGH | CERTAIN | RESOLVED (2026-03-30) — POST /api/outbound with HMAC |
| R28 | `WEBHOOK_SECRET` not required | HIGH | POSSIBLE | RESOLVED (2026-03-30) — validate:"required" tag |
| R29 | `/metrics` unauthenticated | HIGH | POSSIBLE | RESOLVED (2026-03-30) — separate admin port 127.0.0.1:9090 |
| R30 | Plaintext credentials not migrated | HIGH | LIKELY | RESOLVED (2026-03-30) — ConnectionStore with AES-256-GCM encrypt/decrypt, migration 004 |
| R31 | Queue worker silent UPDATE failure | HIGH | POSSIBLE | RESOLVED (2026-03-30) — mark functions return error, abort tx |
| R32 | `sf_connection_id` column name wrong | MEDIUM | CERTAIN | RESOLVED (2026-03-30) — renamed to sf_channel_id |
| R33 | No migration version tracking | MEDIUM | LIKELY | RESOLVED (2026-03-30) — schema_migrations table + version check |
| R34 | Pool closer race on shutdown | MEDIUM | POSSIBLE | RESOLVED (2026-03-30) — pool closed after g.Run() via defer |
| R35 | Unsanitized filename in R2 key | MEDIUM | POSSIBLE | RESOLVED (2026-03-30) — sanitizeKeySegment on all paths |
| R36 | `context.Canceled` exits code 1 | MEDIUM | CERTAIN | RESOLVED (2026-03-30) — graceful exit 0 on SIGINT |
| R37 | Ingester TOCTOU in batch rescue | MEDIUM | POSSIBLE | RESOLVED (2026-03-30) — lock sequence documented + tested |
| R38 | `Authenticate()` missing context | MEDIUM | LIKELY | RESOLVED (2026-03-30) — ctx parameter + NewRequestWithContext |
