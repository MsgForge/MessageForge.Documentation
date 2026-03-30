# MessageForge Implementation Roadmap — Agent-Executable

**Created:** 2026-03-29
**Purpose:** Break the gap between documentation and implementation into discrete phases that agents can execute on command.
**Companion docs:** [Architecture Review](./2026-03-29-architecture-review.md) | [Risk Register](./2026-03-29-risk-register.md)

---

## Overview

The roadmap is organized into 6 phases. Each phase is self-contained and produces a testable deliverable. Phases 1-2 are sequential (foundation). Phases 3-5 can be parallelized across agents. Phase 6 is final integration.

```
Phase 0: Cleanup & ADRs (documentation fixes, remove stale references)
    │
Phase 1: Salesforce Foundation (v11 data model, Protected CMTs, sharing)
    │
Phase 1.5: Critical Data Flow Fixes (NEW 2026-03-30 — ingester fields, chat ID mapping, nil guard, JWT aud, outbound endpoint)
    │
    ├── Phase 2: Go Backend Wiring (connect nil deps, queue workers, MTProto)
    │       │
    │       └── Phase 2.5: Go Quality Fixes (NEW 2026-03-30 — config, transactions, migrations, shutdown)
    │
    ├── Phase 3: Security Hardening (DAST, TLS, metrics auth, credentials encryption, govulncheck)
    │
    └── Phase 4: Infrastructure (deployment, monitoring, HA, proxies)
    │
Phase 5: Integration & Polish (E2E testing, outbound pipeline, delivery sync)
    │
Phase 6: AppExchange Packaging (2GP, test coverage, security review submission)
```

---

## Phase 0: Cleanup & Documentation Fixes

**Goal:** Remove inconsistencies, create missing ADRs, fix stale references.
**Duration:** 1-2 days
**Agent:** Human review + doc-updater
**Risks resolved:** R10, R12, R17

### Tasks

#### 0.1: Replace BC Wallet Style Guide

**Command:** Replace `MessageForge.Documentation/reference/style_guide_cursor_rules.md` with MessageForge-specific Go conventions.

**Source:** Extract patterns from:
- `.claude/agents/go-backend.md` (authoritative)
- `MessageForge.Backend/CLAUDE.md`
- `docs/claude/conventions.md`

**Content to include:**
- oklog/run lifecycle (NOT squad/v2)
- Local interfaces (lowercase, no interfaces.go)
- Constructors at end of file
- Error wrapping (fmt.Errorf, never log+return)
- File naming (plural: handlers.go)
- Config via struct tags (caarlos0/env + validator)
- slog JSON logging
- net/http (no frameworks)

**Acceptance:** File is in English, references MessageForge components, no blockchain/TON/gRPC defaults.

---

#### 0.2: Create ADR-18 — Session Durability Strategy

**File:** `MessageForge.Documentation/architecture/adr.md` (append)

**Decision:** UNLOGGED tables with periodic LOGGED backup snapshot (every 5 minutes). On crash, restore from backup. Maximum data loss: 5 minutes of session state. Users see brief reconnection, not full re-auth.

**Rationale:** UNLOGGED gives 10-50x write performance for high-frequency session updates. Backup snapshot provides crash recovery with bounded data loss.

**Alternatives rejected:**
- Full WAL: too slow for session update frequency
- Accept total loss: unacceptable UX at scale (100+ sessions re-auth)

---

#### 0.3: Create ADR-19 — Data Flow Routing Matrix

**File:** `MessageForge.Documentation/architecture/adr.md` (append)

**Decision:** Clear API assignment per data flow:

| Data Flow | API | Rationale |
|---|---|---|
| Inbound messages (real-time) | Platform Events | Trigger-driven record creation |
| Session status | Platform Events | Infrequent, critical |
| Delivery status | Platform Events | Real-time UI feedback |
| Outbound messages | REST (from Queueable) | Request-response |
| Bulk archive | Bulk API 2.0 | High volume async |
| Channel config read | REST (query) | One-time at startup |
| High-throughput streaming (future) | Pub/Sub API (gRPC) | When PE limits hit |

---

#### 0.4: Fix JWT TTL Documentation

**Where:** `MessageForge.Documentation/reference/security-checklist.md`

**Add clarification table:**

| Token | TTL | System |
|---|---|---|
| Centrifugo JWT | 15 min | LWC -> Centrifugo WebSocket |
| Salesforce OAuth | 90 min | Go -> Salesforce REST API |
| R2 Presigned URL (inbound) | 60 min | Cloudflare Worker -> R2 |
| R2 Presigned URL (outbound) | 15 min | LWC -> R2 PUT |

---

## Phase 1: Salesforce Foundation

**Goal:** Implement complete v11 data model. Create all objects, Protected CMTs, permission sets, sharing model.
**Duration:** 2-3 weeks
**Agent:** `salesforce-dev`
**Risks resolved:** R01, R02, R09
**Prerequisite:** Phase 0

### Tasks

#### 1.1: Create Messenger_Channel__c (replace Messenger_Connection__c)

**Agent prompt:**
```
Create Messenger_Channel__c custom object per sf-data-model-v11.md specification.
Include all 13 fields: Channel_Type_API_Name__c, Display_Name__c, Status__c,
Session_Status__c, Session_Error_Detail__c, Session_Last_Heartbeat__c, Active__c,
Is_Compromised__c, External_Account_ID__c, Go_Connection_Ref__c, Last_Connected_At__c,
Credential_Rotated_At__c. Remove Messenger_Connection__c after migration.
Update ALL Apex classes and LWC components that reference Messenger_Connection__c.
```

**Acceptance criteria:**
- [ ] Object created with all fields
- [ ] All Apex references updated
- [ ] All LWC references updated
- [ ] Tests pass: `sf apex run test --test-level RunLocalTests --wait 30`

---

#### 1.2: Create Platform App Objects (5)

**Objects:** Telegram_App__c, WhatsApp_App__c, Twilio_App__c, Vonage_App__c, Bird_App__c

**Agent prompt:**
```
Create all 5 Platform App objects per sf-data-model-v11.md. Each has account-level
credentials (encrypted where > 175 chars). Include Lookup to User (Manager).
Telegram_App__c: api_id (Text), api_hash (Text Encrypted).
Follow the same pattern for WhatsApp, Twilio, Vonage, Bird per v11 spec.
```

---

#### 1.3: Create Platform Config Objects (8)

**Objects:** Telegram_Bot_Config__c, Telegram_User_Config__c, WhatsApp_Config__c, Viber_Config__c, Twilio_Config__c, Vonage_Config__c, Bird_Config__c, Apple_Messages_Config__c

**Agent prompt:**
```
Create all 8 Platform Config objects per sf-data-model-v11.md. Each is 1:1
Master-Detail to Messenger_Channel__c. Include Lookup to corresponding App object
where applicable. Use Text(Encrypted) for secrets ≤175 chars. Use Apex AES-256
for secrets > 175 chars (WhatsApp Access Token, Vonage Private Key).
```

---

#### 1.4: Create Protected Custom Metadata Types

**Objects:** tgint__Encryption_Key__mdt, tgint__Apple_MSP__mdt

**Agent prompt:**
```
Create 2 Protected Custom Metadata Types per v11 spec. Encryption_Key__mdt:
Key_Value__c (Text 255), Is_Active__c (Checkbox). Apple_MSP__mdt: MSP_ID__c,
MSP_Secret__c, Endpoint_URL__c. Mark both as Protected (visibility: package only).
Migrate HMAC_Secret and Centrifugo_HMAC_Secret from Middleware_Config__mdt to
Encryption_Key__mdt. Update HMACValidator.cls and CentrifugoTokenController.cls.
Delete Middleware_Config__mdt.
```

---

#### 1.5: Create Channel_Type__mdt

**Agent prompt:**
```
Create Channel_Type__mdt Custom Metadata Type per v11 spec. Package-deployed,
read-only for customer. Fields: DeveloperName, Messenger__c, Protocol__c,
Config_Object_Name__c, Supports_Outbound__c. Create records for all 8 channel
types: Telegram_Bot, Telegram_User, WhatsApp_Cloud, Viber_Bot, Twilio_SMS,
Vonage_SMS, Bird_SMS, Apple_Messages.
```

---

#### 1.6: Create Remaining Custom Objects

**Objects:** Channel_User_Access__c, Messenger_Template__c, Messenger_Attachment__c, Channel_Audit_Log__c

**Agent prompt:**
```
Create 4 remaining custom objects per v11 spec:
- Channel_User_Access__c: MD to Channel, polymorphic (User or Group), access levels
- Messenger_Template__c: Lookup to Channel, template body with placeholders
- Messenger_Attachment__c: MD to Message, multi-media support, sort order
- Channel_Audit_Log__c: Lookup to Channel (survives deletion), event tracking
Update Messenger_Message__c: change Chat__c from Lookup to Master-Detail.
Add Template__c Lookup and Attachment_Count__c rollup.
```

---

#### 1.7: Implement Sharing Model

**Agent prompt:**
```
Implement Apex Managed Sharing for Messenger_Chat__c:
1. Set OWD to Private for Messenger_Chat__c
2. Create ChannelAccessService.cls — recalculate sharing based on Channel_User_Access__c
3. Create ChannelAccessTrigger.trigger — fire on Channel_User_Access__c insert/update/delete
4. Test with multi-user scenario: Agent A sees only their chats, Agent B sees only theirs
```

---

#### 1.8: Create Messenger_Viewer Permission Set

**Agent prompt:**
```
Create tgint__Messenger_Viewer permission set per v11 spec. Read-only access to:
Messenger_Channel__c, Messenger_Chat__c, Messenger_Message__c, Messenger_Attachment__c,
Channel_Audit_Log__c. No access to Config, App, or Access objects.
```

---

#### 1.9: Update Platform Events to v11

**Agent prompt:**
```
Update Inbound_Message__e: rename Chat_SF_ID__c, add Media_MIME_Type__c field.
Update Session_Status__e: rename Connection_SF_ID__c to Channel_SF_ID__c.
Update all trigger handlers to use new field names.
Run all tests.
```

---

## Phase 1.5: Critical Data Flow Fixes (NEW — 2026-03-30)

**Goal:** Fix the 3 critical and 2 high issues that block any integration testing.
**Duration:** 1-2 days
**Agent:** `go-backend` + `salesforce-dev` (parallel)
**Risks resolved:** R23, R24, R25, R26, R27
**Prerequisite:** Phase 0
**Added by:** [2026-03-30 Recheck Review](./2026-03-30-recheck-review.md)

### Tasks

#### 1.5.1: Fix Ingester Payload Field Names (R24)

**Agent:** `go-backend`
**File:** `MessageForge.Backend/internal/salesforce/ingestion.go:38-49`

**Current (broken):**
```
External_ID__c, Platform__c, Chat_ID__c, Sender_ID__c, Media_Type__c, Media_File_ID__c
```

**Required (match Inbound_Message__e):**
```
Text__c, Chat_SF_ID__c, Sender_Name__c, Sender_External_ID__c, Message_Type__c, Media_URL__c, Media_MIME_Type__c, Sent_At__c, Protocol__c
```

**Acceptance:** Ingester payload keys match `Inbound_Message__e` field API names exactly.

---

#### 1.5.2: Add Chat External ID Resolution (R25)

**Agent:** `salesforce-dev`
**File:** `MessageForge.Salesforce/force-app/main/default/classes/InboundMessageTriggerHandler.cls`

**Approach (Option B — Apex-side lookup):**
1. Accept `Chat_External_ID__c` (Telegram chat ID) instead of requiring `Chat_SF_ID__c`
2. In trigger handler, query `Messenger_Chat__c` by `Chat_External_ID__c`
3. If not found, auto-create `Messenger_Chat__c` record
4. Link message to resolved chat record

**Acceptance:** Messages with Telegram chat IDs are persisted in Salesforce.

---

#### 1.5.3: Guard Nil Media Downloader (R23)

**Agent:** `go-backend`
**File:** `MessageForge.Backend/internal/messenger/app.go:205`

**Fix:** When downloader is nil, pass `nil` for `mediaProc` (not a constructed-with-nil processor):
```go
// Before: mediaProc = media.NewInboundProcessor(nil, r2Client, 60*time.Minute)
// After:  mediaProc remains nil when downloader is unavailable
```

Pipeline already guards `p.media != nil` — this fix makes the guard effective.

**Acceptance:** Media messages are skipped gracefully, no panic.

---

#### 1.5.4: Fix JWT `aud` Claim (R26)

**Agent:** `go-backend`
**File:** `MessageForge.Backend/internal/salesforce/auth.go:93`

**Fix:** Change `"aud": a.config.TokenURL` to `"aud": "https://login.salesforce.com"` (or extract base URL from TokenURL for sandbox support).

**Acceptance:** Go middleware successfully authenticates to Salesforce via JWT Bearer Flow.

---

#### 1.5.5: Register Outbound Endpoint (R27)

**Agent:** `go-backend`
**File:** `MessageForge.Backend/internal/messenger/app.go`

**Fix:** Register `/api/outbound` handler with HMAC middleware. Wire to insert into `outbound_queue` table for async processing by `OutboundWorker`.

**Acceptance:** Salesforce `MessengerOutboundService` can POST to `/api/outbound` and receive 200.

---

## Phase 2: Go Backend Wiring

**Goal:** Connect all nil dependencies, implement queue workers, port MTProto from PoC.
**Duration:** 3-4 weeks
**Agent:** `go-backend`
**Risks resolved:** R04 (partial)
**Prerequisite:** Phase 0, Phase 1, **Phase 1.5 (NEW)**

### Tasks

#### 2.1: Wire Database Pool into App

**Agent prompt:**
```
In MessageForge.Backend/internal/messenger/app.go, create database pool from config
and pass to all services that need it. Currently NewPipeline receives nil for
database-dependent components. Wire:
- database.NewPool(cfg.DatabaseURL) in NewApp()
- Pass pool to session store, queue workers, media tracker
- Execute migrations on startup (database.Migrate)
- Add pool health check to /health endpoint
```

---

#### 2.2: Implement Inbound Queue Worker

**Agent prompt:**
```
Create internal/messenger/queue_worker.go. Implement PostgreSQL-based queue worker
using FOR UPDATE SKIP LOCKED pattern (from postgres-schema.md). Worker:
1. Polls inbound_queue for pending messages
2. Calls Salesforce ingestion (Platform Event publish)
3. Updates status to 'delivered' or 'failed'
4. Handles retries with exponential backoff
5. Integrates with oklog/run as an actor (execute/interrupt pair)
```

---

#### 2.3: Implement Outbound Queue Worker

**Agent prompt:**
```
Create internal/messenger/outbound_worker.go. Processes outbound_queue:
1. Polls for pending outbound messages (SKIP LOCKED)
2. Routes to correct adapter (Telegram Bot API for now)
3. Sends message via Telegram
4. Updates status + publishes delivery status Platform Event
5. Handles terminal failures (USER_BANNED, CHAT_WRITE_FORBIDDEN)
6. Exponential backoff with max 5 retries
Wire into app.go with oklog/run actor.
```

---

#### 2.4: Wire Media Processor into Pipeline

**Agent prompt:**
```
In app.go, wire inbound media processor (media.NewInboundProcessor) into pipeline.
Currently nil. The processor should:
1. Check if message has MediaFileID
2. Download from Telegram (Bot API: getFile endpoint)
3. Upload to R2 via media.R2Client
4. Set MediaURL on message
5. Store reference in media_files table
Pass R2 client + database pool to media processor constructor.
```

---

#### 2.5: Wire Salesforce Ingester into Pipeline

**Agent prompt:**
```
In app.go, wire Salesforce batch ingester into pipeline. Currently nil.
Create salesforce client from config (JWT auth), create batch ingester,
pass to pipeline. Ensure ingester starts its flush loop as oklog/run actor.
```

---

#### 2.6: Port MTProto Client from PoC

**Agent prompt:**
```
Port MTProto client from MessageForge.PoC/internal/telegram/mtproto.go to
MessageForge.Backend/internal/telegram/mtproto/client.go. Adapt:
- Use database session store instead of in-memory
- Integrate with rate limiting middleware chain
- Implement as MessengerAdapter interface
- Add to oklog/run lifecycle
- Wire into app.go alongside Bot API handler
```

---

#### 2.7: Implement Session Backup Mechanism (ADR-18)

**Agent prompt:**
```
Create internal/database/session_backup.go. Implement periodic snapshot:
1. Every 5 minutes, copy mtproto_sessions to mtproto_sessions_backup (LOGGED table)
2. On startup, if mtproto_sessions is empty, restore from backup
3. Log backup events with slog
4. Add as oklog/run actor
Create migration 002_session_backup.sql for backup table.
```

---

## Phase 2.5: Go Quality Fixes (NEW — 2026-03-30)

**Goal:** Fix high/medium Go issues discovered in recheck that don't block integration but affect reliability.
**Duration:** 1-2 days
**Agent:** `go-backend`
**Risks resolved:** R28, R31, R32, R33, R34, R36, R37, R38
**Prerequisite:** Phase 1.5
**Added by:** [2026-03-30 Recheck Review](./2026-03-30-recheck-review.md)

### Tasks

#### 2.5.1: Make `WEBHOOK_SECRET` Required (R28)

**File:** `internal/messenger/config/config.go:41`
**Fix:** Add `validate:"required"` tag to `WebhookSecret`.

---

#### 2.5.2: Abort Transaction on Mark Failure (R31)

**Files:** `internal/messenger/queue_worker.go`, `internal/messenger/outbound_worker.go`
**Fix:** Return error from `markDelivered`/`markFailed`/`markOutboundSent`/`markOutboundFailed`. On error, rollback transaction instead of committing.

---

#### 2.5.3: Rename `sf_connection_id` to `sf_channel_id` (R32)

**File:** `internal/database/migrations/001_initial.sql:9`
**Fix:** Rename column. Update all Go code referencing this column.

---

#### 2.5.4: Add Migration Version Tracking (R33)

**File:** `internal/database/migrations.go`
**Fix:** Create `schema_migrations` table. Before executing each migration, check if version already applied. Record applied version after execution.

---

#### 2.5.5: Fix Pool Closer Shutdown Ordering (R34)

**File:** `internal/messenger/app.go:74`
**Fix:** Remove pool close from `oklog/run` actor. Close pool with `defer` after `g.Run()` returns in `main.go`.

---

#### 2.5.6: Handle `context.Canceled` Gracefully (R36)

**File:** `cmd/messenger/main.go:47-49`
**Fix:** Check `errors.Is(err, context.Canceled)` — if true, log info and exit 0.

---

#### 2.5.7: Fix Ingester TOCTOU Window (R37)

**File:** `internal/salesforce/ingestion.go:53-76`
**Fix:** Keep mutex held through rescue + append sequence. Single critical section.

---

#### 2.5.8: Add `context.Context` to `Authenticate()` (R38)

**File:** `internal/salesforce/auth.go:46`
**Fix:** Add `ctx context.Context` parameter. Use `http.NewRequestWithContext`. Update all callers.

---

## Phase 3: Security Hardening

**Goal:** Complete all security checklist items. Prepare for DAST scan.
**Duration:** 1-2 weeks (can parallel with Phase 2)
**Agent:** `security-reviewer`
**Risks resolved:** R02, R03, R29, R30

### Tasks

#### 3.1: OWASP ZAP Scan Setup

**Steps:**
1. Deploy Go middleware to Docker (use existing docker-compose)
2. Install OWASP ZAP (Docker: `owasp/zap2docker-stable`)
3. Configure target: `http://localhost:8080`
4. Run automated scan
5. Fix all HIGH/CRITICAL findings
6. Generate PDF report

---

#### 3.2: TLS Configuration

**Steps:**
1. Document Caddy/nginx reverse proxy config for TLS termination
2. Configure Let's Encrypt auto-renewal
3. Run SSL Labs scan, save PDF
4. Enforce HSTS headers

---

#### 3.3: Complete Security Checklist

**Unchecked items to resolve:**
- [ ] R2 bucket strictly private — verify bucket policy, disable public access
- [ ] Presigned URL TTL capped to 60 min — configure in R2 client constructor
- [ ] R2 CORS policy — configure for Salesforce instance domains
- [ ] CSP Trusted Sites (3 items) — Centrifugo WS, R2 endpoint, Worker CDN
- [ ] Bot tokens encrypted at rest — add AES encryption to credentials column

---

#### 3.4: Encrypt Bot Tokens at Rest (R30)

**Agent prompt:**
```
AES-256-GCM implementation exists in database/crypto.go but is not wired.
Wire crypto.Encrypt/crypto.Decrypt into connection repository read/write path.
Migrate data from plaintext credentials JSONB to encrypted_credentials BYTEA.
Drop or nullify the plaintext column after migration.
```

---

#### 3.5: Protect `/metrics` Endpoint (R29) (NEW — 2026-03-30)

**Agent:** `go-backend`
**File:** `MessageForge.Backend/internal/server/server.go:92`

**Options (pick one):**
- A) Move `/metrics` to a separate admin port (e.g., `:9090`) — **recommended**
- B) Add bearer token authentication middleware
- C) Restrict to localhost via IP check middleware

**Acceptance:** `/metrics` not accessible from public internet without authentication.

---

#### 3.6: Sanitize R2 Key Paths (R35) (NEW — 2026-03-30)

**Agent:** `go-backend`
**File:** `MessageForge.Backend/internal/media/outbound.go:58`

**Fix:** Strip path separators and `..` from `req.Filename` and `req.ChatID` before constructing R2 key. Use `filepath.Base()` or a whitelist of safe characters.

---

#### 3.7: Run `govulncheck` and `npm audit` (NEW — 2026-03-30)

**Steps:**
1. Run `govulncheck ./...` in MessageForge.Backend
2. Run `npm audit` in MessageForge.Backend/workers/media-proxy
3. Fix any HIGH/CRITICAL vulnerabilities
4. Document results

---

## Phase 4: Infrastructure

**Goal:** Production deployment guide, monitoring, HA basics.
**Duration:** 1-2 weeks (can parallel with Phase 2-3)
**Agent:** Human + documentation
**Risks resolved:** R05, R06, R13, R14

### Tasks

#### 4.1: Create Deployment Guide

**File:** `MessageForge.Documentation/reference/deployment-guide.md`

**Content:**
- Hetzner VPS provisioning (ARM Ampere, Ubuntu 24.04)
- systemd unit files for: Go middleware, Centrifugo
- PostgreSQL setup (managed or self-hosted)
- Caddy reverse proxy with auto-TLS
- Cloudflare R2 bucket creation + Worker deployment
- Environment variable template
- First-run checklist

---

#### 4.2: Implement Monitoring

**In Go middleware:**
- Add `/metrics` endpoint (Prometheus format)
- Key metrics: messages_processed_total, messages_failed_total, queue_depth, telegram_flood_waits, salesforce_token_refreshes, request_duration_seconds

**External:**
- UptimeRobot for /health endpoint
- Grafana dashboard template

---

#### 4.3: Research and Select Proxy Provider

**Evaluate:**
| Provider | Telegram Compatible | Price/IP/month | Notes |
|---|---|---|---|
| Bright Data | Verify | $15-25 | Market leader |
| IPRoyal | Verify | $5-8 | Budget option |
| SmartProxy | Verify | $12-14 | EU coverage |
| Oxylabs | Verify | $15-20 | Enterprise |

**Test:** 30-day trial with 5 IPs, running MTProto sessions through proxy. Track: ban rate, latency, uptime.

---

#### 4.4: Create Cost Model

**File:** `MessageForge.Documentation/reviews/cost-model.md`

**Cover:** Infrastructure costs at 3 tiers: MVP (10 channels), Growth (100 channels), Scale (500 channels). Include: VPS, DB, proxies, R2, Workers, domains.

---

## Phase 5: Integration & Polish

**Goal:** End-to-end message flow, outbound pipeline, delivery sync.
**Duration:** 2-3 weeks
**Agent:** `go-backend` + `salesforce-dev` (parallel)
**Prerequisite:** Phases 1-4

### Tasks

#### 5.1: E2E Inbound Flow Test

**Test:** Send Telegram message -> Go receives -> pipeline processes -> SF Platform Event -> Apex trigger creates record -> Centrifugo publishes -> LWC displays.

**Verify:** Message appears in Salesforce within 5 seconds. Centrifugo delivers real-time update.

---

#### 5.2: E2E Outbound Flow Test

**Test:** Agent types in LWC -> Apex Queueable fires -> REST to Go -> Go sends via Telegram -> Delivery status Platform Event -> LWC updates status.

**Verify:** Message delivered to Telegram. Status shows "delivered" in LWC.

---

#### 5.3: Implement channelSetupWizard LWC

**Agent prompt (salesforce-dev):**
```
Create channelSetupWizard LWC per v11 spec. Guided channel creation:
1. Select platform (from Channel_Type__mdt)
2. Select or create App object
3. Create Channel + Config objects
4. Save credentials (encrypted)
5. Trigger Go middleware connection via REST callout
```

---

#### 5.4: Implement ChannelCompromiseTrigger

**Agent prompt (salesforce-dev):**
```
Create ChannelCompromiseTrigger.trigger on Messenger_Channel__c. When
Is_Compromised__c is set to true:
1. Set Status__c to 'suspended'
2. Set Active__c to false
3. REST callout to Go middleware to disconnect session
4. Create Channel_Audit_Log__c record (event_type: 'compromised')
```

---

#### 5.5: Implement Delivery Status Sync (Go side)

**Agent prompt (go-backend):**
```
In internal/salesforce/delivery_status.go, implement delivery status publisher.
When outbound message terminal state reached (delivered, failed):
1. Publish tgint__Message_Delivery_Status__e Platform Event
2. Include: Message_External_ID, Status, Error_Code, Error_Detail, Timestamp
3. Batch delivery status events (same sync.Pool pattern as ingestion)
```

---

## Phase 6: AppExchange Packaging

**Goal:** Package, test coverage, security review submission.
**Duration:** 2-4 weeks
**Agent:** `salesforce-dev` + `security-reviewer`
**Prerequisite:** Phase 5
**Risks resolved:** R03

### Tasks

#### 6.1: Verify Test Coverage

```bash
sf apex run test --test-level RunLocalTests --code-coverage --wait 30
```

**Target:** 90%+ overall. 75% minimum (AppExchange requirement).

**Missing test classes to create:**
- ChannelAccessServiceTest.cls
- ChannelCompromiseTriggerHandlerTest.cls
- ChannelSetupControllerTest.cls
- EncryptionServiceTest.cls

---

#### 6.2: Create 2GP Package Version

```bash
sf package version create --package MessageForge --installation-key-bypass --wait 30
```

**Verify:** Install in fresh scratch org, run all tests, verify all objects/fields/LWC/triggers deploy.

---

#### 6.3: Partner Community Enrollment

**Manual steps:**
1. Register at Salesforce Partner Community
2. Complete ISV application
3. Request Dev Hub access
4. Set up Partner Business Org

---

#### 6.4: Prepare Security Review Submission

**Deliverables:**
- [ ] DAST scan PDF (zero high-severity)
- [ ] SSL Labs scan PDF (TLS 1.2+ verified)
- [ ] Architecture diagram (Composite App)
- [ ] Data flow documentation
- [ ] Test scratch org with sample data
- [ ] Connected App configuration
- [ ] PADA signed

---

#### 6.5: Submit Security Review

**Manual:** Submit through Partner Community portal with all deliverables.

**Expected timeline:** 4-8 weeks for review.

---

## Execution Quick Reference

### Phase Dependencies (Updated 2026-03-30)

```
Phase 0 ─────────────────────────────────────────────────────►
         Phase 1 (SF) ──────────────────────────────────────►
                   Phase 1.5 (Data Flow Fixes) ─────────────► NEW
                        Phase 2 (Go Wiring) ────────────────►
                        Phase 2.5 (Go Quality) ────────────► NEW
                        Phase 3 (Sec) ──────────────────────►
                        Phase 4 (Infra) ────────────────────►
                                             Phase 5 ───────►
                                                     Phase 6 ──►
```

**Critical path:** Phase 0 -> Phase 1 -> Phase 1.5 -> Phase 2 -> Phase 5 -> Phase 6

### Agent Assignment

| Phase | Primary Agent | Secondary | Duration |
|---|---|---|---|
| 0 | doc-updater | Human review | 1-2 days |
| 1 | salesforce-dev | security-reviewer | 2-3 weeks |
| 1.5 (NEW) | go-backend + salesforce-dev | — | 1-2 days |
| 2 | go-backend | — | 3-4 weeks |
| 2.5 (NEW) | go-backend | — | 1-2 days |
| 3 | security-reviewer | go-backend | 1-2 weeks |
| 4 | Human | doc-updater | 1-2 weeks |
| 5 | go-backend + salesforce-dev | e2e-runner | 2-3 weeks |
| 6 | salesforce-dev | security-reviewer | 2-4 weeks |

### Per-Phase Verification Commands

```bash
# Phase 1: Salesforce
cd MessageForge.Salesforce && sf apex run test --test-level RunLocalTests --code-coverage --wait 30

# Phase 1.5: Data Flow Fixes
cd MessageForge.Backend && go vet ./... && go test -race ./... -count=1
# Then: manual integration test — send Telegram message, verify SF record created

# Phase 2: Go Backend
cd MessageForge.Backend && go vet ./... && go test -race ./... -count=1

# Phase 2.5: Go Quality
cd MessageForge.Backend && go vet ./... && go test -race ./... -count=1

# Phase 3: Security
cd MessageForge.Backend && govulncheck ./...
# OWASP ZAP: docker run -t owasp/zap2docker-stable zap-baseline.py -t http://localhost:8080
# SSL Labs: https://www.ssllabs.com/ssltest/

# Phase 5: E2E
# Manual: send Telegram message, verify in SF, verify in LWC

# Phase 6: Package
sf package version create --package MessageForge --installation-key-bypass --wait 30
```

---

## Success Criteria

| Metric | Target |
|---|---|
| SF Objects Created | 21/21 + 2 Protected CMTs |
| Go Test Coverage | 80%+ |
| Apex Test Coverage | 90%+ (75% minimum) |
| Security Checklist | All items checked |
| DAST Findings | Zero HIGH/CRITICAL |
| E2E Inbound Latency | < 5 seconds |
| E2E Outbound Latency | < 10 seconds |
| Session Recovery | < 30 seconds after crash |
| Uptime (with systemd) | 99.5%+ |
