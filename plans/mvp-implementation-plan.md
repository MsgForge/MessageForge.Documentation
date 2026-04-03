# MVP Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.
> **Workflow:** Use `dev-review-loop` skill for each task. Use `question-resolver` agent for decisions — do NOT ask the user.
> **Notify user:** Only at phase completion or when genuinely stuck.

**Goal:** Build a working Messenger-to-Salesforce integration with Telegram (Bot API + MTProto), live chat UI in Salesforce Lightning, and media handling via Salesforce ContentVersion.

**Architecture:** Go middleware connects Telegram to Salesforce via REST/Platform Events. PostgreSQL stores sessions and queues. Platform Events via `lightning/empApi` provide real-time updates to LWC. Media stored in Salesforce ContentVersion (Files).

**Tech Stack:** Go 1.23+, gotd/td, GoTGProto, pgx/v5, Salesforce Apex/LWC (2GP, namespace: tgint__)

---

## Phase 1: Foundation

**Agent:** `go-backend`
**Skills:** None required (basic scaffolding)
**Depends on:** Nothing

### Task 1.1: Go Module & Dependencies

**Files:**
- Modify: `go.mod`
- Create: `go.sum` (auto-generated)

**Step 1:** Run `go mod tidy` to resolve and download all dependencies
**Step 2:** Verify: `go build ./...` compiles without errors
**Step 3:** Commit: `feat: initialize go module with dependencies`

---

### Task 1.2: Configuration with Validation

**Files:**
- Modify: `internal/config/config.go`
- Create: `internal/config/config_test.go`

**Step 1: Write failing test**
```go
func TestLoad_RequiresDatabaseURL(t *testing.T) {
    t.Setenv("DATABASE_URL", "")
    _, err := Load()
    if err == nil { t.Fatal("expected error for missing DATABASE_URL") }
}

func TestLoad_Defaults(t *testing.T) {
    t.Setenv("DATABASE_URL", "postgres://localhost/test")
    cfg, err := Load()
    if err != nil { t.Fatal(err) }
    if cfg.Port != "8080" { t.Errorf("expected default port 8080, got %s", cfg.Port) }
}
```

**Step 2:** Run: `go test ./internal/config/ -v` — expect PASS (existing impl covers this)
**Step 3:** Add Telegram API ID parsing (string→int) with validation
**Step 4:** Run tests again, verify pass
**Step 5:** Commit: `feat: config loading with validation and tests`

---

### Task 1.3: PostgreSQL Connection Pool

**Files:**
- Create: `internal/database/pool.go`
- Create: `internal/database/pool_test.go`

**Skill:** Read `MessageForge.Documentation/reference/postgres-schema.md`

**Step 1: Write failing test**
```go
func TestNewPool_InvalidURL(t *testing.T) {
    _, err := NewPool(context.Background(), "invalid://url")
    if err == nil { t.Fatal("expected error") }
}
```

**Step 2:** Implement `NewPool(ctx, url) (*pgxpool.Pool, error)` using pgx/v5
**Step 3:** Run: `go test ./internal/database/ -v`
**Step 4:** Commit: `feat: postgresql connection pool with pgx/v5`

---

### Task 1.4: Database Schema Migration

**Files:**
- Create: `internal/database/migrate.go`
- Create: `internal/database/migrations/001_initial.sql`
- Create: `internal/database/migrate_test.go`

**Skill:** `MessageForge.Documentation/reference/postgres-schema.md` — copy exact schema

**Step 1:** Create migration SQL from postgres-schema.md (connections, sessions, inbound_queue, outbound_queue tables)
**Step 2:** Implement `Migrate(ctx, pool)` — executes SQL files in order
**Step 3:** Test requires running PostgreSQL: `docker compose up -d postgres`
**Step 4:** Write integration test with test database
**Step 5:** Run: `go test ./internal/database/ -v -tags=integration`
**Step 6:** Commit: `feat: database schema migration with initial tables`

---

### Task 1.5: HTTP Server Skeleton

**Files:**
- Create: `internal/server/server.go`
- Create: `internal/server/server_test.go`
- Create: `internal/server/middleware.go`
- Modify: `cmd/messenger/main.go`

**Step 1: Write failing test**
```go
func TestHealthEndpoint(t *testing.T) {
    srv := New(Config{Port: "0"})
    // test GET /health returns 200
}
```

**Step 2:** Implement HTTP server with `/health` endpoint using `net/http`
**Step 3:** Add HMAC validation middleware (for future webhook endpoints)
**Step 4:** Wire into `cmd/messenger/main.go` with graceful shutdown
**Step 5:** Run: `go test ./internal/server/ -v`
**Step 6:** Run: `go build ./cmd/messenger/ && ./messenger` — verify health endpoint works
**Step 7:** Commit: `feat: http server with health endpoint and hmac middleware`

---

### Task 1.6: Bot API Webhook Receiver

**Files:**
- Create: `internal/telegram/botapi/handler.go`
- Create: `internal/telegram/botapi/handler_test.go`

**Skill:** `rate-limiting` (Bot API limits section)

**Step 1: Write failing test** — POST /webhook/bot with valid Telegram update JSON
**Step 2:** Implement webhook handler that parses Telegram Bot API updates
**Step 3:** Convert to `adapter.Message` format
**Step 4:** Channel the message to inbound pipeline
**Step 5:** Run: `go test ./internal/telegram/botapi/ -v`
**Step 6:** Commit: `feat: telegram bot api webhook receiver`

---

**Phase 1 complete → notify user with summary**

---

## Phase 2: Core MTProto

**Agent:** `go-backend`
**Skills:** `telegram-auth-flow`, `rate-limiting`
**Depends on:** Phase 1

### Task 2.1: MTProto Client Wrapper

**Files:**
- Create: `internal/telegram/mtproto/client.go`
- Create: `internal/telegram/mtproto/client_test.go`

**Skill:** `telegram-auth-flow` — MUST invoke before starting

**Step 1:** Implement `Client` struct wrapping `gotd/td` via GoTGProto
**Step 2:** Add Connect/Disconnect methods matching `adapter.Adapter` interface
**Step 3:** Test with mock (no real Telegram connection needed)
**Step 4:** Commit: `feat: mtproto client wrapper with gotgproto`

---

### Task 2.2: Auth Flow State Machine

**Files:**
- Create: `internal/telegram/mtproto/auth.go`
- Create: `internal/telegram/mtproto/auth_test.go`

**Skill:** `telegram-auth-flow` — 5-phase state machine

**Step 1:** Define auth states: `PhoneInput → OTPSent → OTPInput → TwoFA → Success`
**Step 2:** Implement `SendCode(phone)` → returns code hash
**Step 3:** Implement `SignIn(phone, code, codeHash)`
**Step 4:** Implement SRP v6a 2FA: `CheckPassword(password)` using gotd/td crypto/srp
**Step 5:** Test each state transition with mocks
**Step 6:** Commit: `feat: mtproto auth flow with srp v6a 2fa`

---

### Task 2.3: Session Persistence

**Files:**
- Create: `internal/telegram/mtproto/session_store.go`
- Create: `internal/telegram/mtproto/session_store_test.go`

**Skill:** `telegram-auth-flow` (session storage section)

**Step 1:** Implement `SessionStore` backed by PostgreSQL UNLOGGED table
**Step 2:** Methods: `Load(userID)`, `Save(userID, data)`, `Delete(userID)`
**Step 3:** Integration test with local PostgreSQL
**Step 4:** Commit: `feat: mtproto session persistence in postgresql`

---

### Task 2.4: Channel Message Parser

**Files:**
- Create: `internal/telegram/mtproto/messages.go`
- Create: `internal/telegram/mtproto/messages_test.go`

**Step 1:** Implement channel message listener (subscribe to updates)
**Step 2:** Parse message content: text, media references, sender info
**Step 3:** Convert to `adapter.Message` format
**Step 4:** Implement `InboundMessages(ctx)` returning `<-chan Message`
**Step 5:** Test with mock updates
**Step 6:** Commit: `feat: mtproto channel message parser`

---

### Task 2.5: Rate Limiting Middleware Chain

**Files:**
- Create: `internal/telegram/mtproto/ratelimit.go`
- Create: `internal/telegram/mtproto/ratelimit_test.go`

**Skill:** `rate-limiting` — middleware chain section

**Step 1:** Implement ratelimit middleware using `golang.org/x/time/rate`
**Step 2:** Implement floodwait middleware (catch FLOOD_WAIT, sleep, retry)
**Step 3:** Chain: ratelimit → floodwait → RPC
**Step 4:** Test with simulated rate limit errors
**Step 5:** Commit: `feat: mtproto rate limiting middleware chain`

---

### Task 2.6: Telegram Adapter (unified interface)

**Files:**
- Create: `internal/telegram/adapter.go`
- Create: `internal/telegram/adapter_test.go`

**Step 1:** Implement `TelegramAdapter` that satisfies `adapter.Adapter`
**Step 2:** Support both MTProto and Bot API connections
**Step 3:** Route operations to appropriate sub-client based on connection type
**Step 4:** Test both paths
**Step 5:** Commit: `feat: unified telegram adapter implementing messenger interface`

---

**Phase 2 complete → notify user with summary**

---

## Phase 3: Salesforce Integration

**Agent:** `salesforce-dev` (Apex/LWC), `go-backend` (OAuth client)
**Skills:** `salesforce-apex-patterns`, `data-ingestion`
**Depends on:** Phase 1 (Go side), independent for SF metadata

### Task 3.1: Custom Objects

**Files:**
- Create: `salesforce/force-app/main/default/objects/Messenger_Connection__c/`
- Create: `salesforce/force-app/main/default/objects/Messenger_Chat__c/`
- Create: `salesforce/force-app/main/default/objects/Messenger_Message__c/`
- Create: `salesforce/force-app/main/default/objects/Messenger_Event__c/`

**Skill:** `salesforce-apex-patterns`, read `MessageForge.Documentation/reference/sf-data-model.md`

**Step 1:** Create object XML metadata for all 4 custom objects with fields per sf-data-model.md
**Step 2:** Deploy to scratch org: `sf project deploy start --source-dir salesforce/force-app`
**Step 3:** Verify: `sf org list` and check objects exist
**Step 4:** Commit: `feat: custom objects for messenger integration`

---

### Task 3.2: Platform Events

**Files:**
- Create: `salesforce/force-app/main/default/objects/Inbound_Message__e/`
- Create: `salesforce/force-app/main/default/objects/Session_Status__e/`
- Create: `salesforce/force-app/main/default/objects/Message_Delivery_Status__e/`

**Skill:** read `MessageForge.Documentation/reference/sf-data-model.md` (Platform Events section)

**Step 1:** Create Platform Event metadata XML for all 3 events
**Step 2:** Deploy and verify
**Step 3:** Commit: `feat: platform events for messenger integration`

---

### Task 3.3: Protected Custom Metadata

**Files:**
- Create: `salesforce/force-app/main/default/customMetadata/Messenger_Settings__mdt/`

**Step 1:** Create Protected Custom Metadata Type for secrets (webhook_secret)
**Step 2:** Deploy and verify
**Step 3:** Commit: `feat: protected custom metadata for secrets`

---

### Task 3.4: Apex REST Endpoint with HMAC

**Files:**
- Create: `salesforce/force-app/main/default/classes/MessengerInboundAPI.cls`
- Create: `salesforce/force-app/main/default/classes/MessengerInboundAPI.cls-meta.xml`
- Create: `salesforce/force-app/main/default/classes/MessengerInboundAPITest.cls`
- Create: `salesforce/force-app/main/default/classes/MessengerInboundAPITest.cls-meta.xml`

**Skill:** `salesforce-apex-patterns` (REST endpoint + HMAC section)

**Step 1:** Write test class first (mock HTTP requests with valid/invalid HMAC)
**Step 2:** Implement `@RestResource(urlMapping='/messenger/inbound/*')`
**Step 3:** HMAC-SHA256 validation using `Crypto.generateMac` + constant-time comparison
**Step 4:** Parse inbound messages, create Messenger_Message__c records (bulkified)
**Step 5:** Deploy and run tests: `sf apex run test --synchronous`
**Step 6:** Commit: `feat: apex rest endpoint with hmac validation`

---

### Task 3.5: Platform Event Triggers

**Files:**
- Create: `salesforce/force-app/main/default/triggers/InboundMessageTrigger.trigger`
- Create: `salesforce/force-app/main/default/triggers/InboundMessageTrigger.trigger-meta.xml`
- Create: `salesforce/force-app/main/default/classes/InboundMessageTriggerHandler.cls`
- Create: `salesforce/force-app/main/default/classes/InboundMessageTriggerHandlerTest.cls`

**Skill:** `salesforce-apex-patterns` (Platform Event Trigger pattern)

**Step 1:** Write test first — publish test events, verify records created
**Step 2:** Implement trigger handler (bulkified insert of Messenger_Message__c)
**Step 3:** Deploy and test
**Step 4:** Commit: `feat: platform event trigger for inbound messages`

---

### Task 3.6: OAuth JWT Bearer Flow (Go side)

**Files:**
- Create: `internal/salesforce/auth.go`
- Create: `internal/salesforce/auth_test.go`
- Create: `internal/salesforce/client.go`
- Create: `internal/salesforce/client_test.go`

**Skill:** `salesforce-apex-patterns` (OAuth section)

**Step 1:** Implement JWT Bearer token generation in Go (RS256 signing)
**Step 2:** Implement token exchange with SF token endpoint
**Step 3:** Implement `SFClient` with auto-refresh
**Step 4:** Test with mock HTTP server
**Step 5:** Commit: `feat: salesforce oauth jwt bearer flow in go`

---

### Task 3.7: Data Ingestion Pipeline

**Files:**
- Create: `internal/salesforce/ingestion.go`
- Create: `internal/salesforce/ingestion_test.go`

**Skill:** `data-ingestion` — MUST invoke

**Step 1:** Implement batch collector with `sync.Pool` for byte buffers
**Step 2:** Sliding window batching (950KB threshold)
**Step 3:** Support Platform Events publish + REST API fallback
**Step 4:** Test batching logic with various payload sizes
**Step 5:** Commit: `feat: salesforce data ingestion with dynamic batching`

---

### Task 3.8: Basic LWC Chat Component

**Files:**
- Create: `salesforce/force-app/main/default/lwc/messengerChat/`
  - `messengerChat.html`
  - `messengerChat.js`
  - `messengerChat.css`
  - `messengerChat.js-meta.xml`

**Skill:** `salesforce-apex-patterns` (LWC section)

**Step 1:** Create LWC with wire adapter to fetch Messenger_Message__c records
**Step 2:** Display message list (sender, text, timestamp)
**Step 3:** Add empApi subscription for Inbound_Message__e (live updates)
**Step 4:** Deploy and verify in scratch org
**Step 5:** Commit: `feat: basic lwc chat component with live updates`

---

**Phase 3 complete → notify user with summary**

---

## Phase 4: Media Pipeline

**Agent:** `go-backend` (ContentVersion upload/download), `salesforce-dev` (LWC rendering)
**Skills:** `media-pipeline`
**Depends on:** Phase 1 + Phase 3

### Task 4.1: Go ContentVersion REST API Client (multipart upload)

**Files:**
- Create: `internal/salesforce/contentversion.go`
- Create: `internal/salesforce/contentversion_test.go`

**Skill:** `media-pipeline` — MUST invoke

**Step 1:** Implement `ContentVersionUploader` using multipart form-data POST to `/services/data/v62.0/sobjects/ContentVersion`
**Step 2:** Methods: `Upload(ctx, params) (ContentVersionResult, error)`, `Download(ctx, contentVersionID) (io.ReadCloser, string, error)`
**Step 3:** Set `FirstPublishLocationId` to auto-create `ContentDocumentLink`
**Step 4:** Query back `ContentDocumentId` from created `ContentVersion`
**Step 5:** Test with mock HTTP server
**Step 6:** Commit: `feat: contentversion multipart upload and download adapter`

---

### Task 4.2: ContentDocumentLink Creation and Trigger

**Files:**
- Create: `salesforce/force-app/main/default/triggers/ContentDocumentLinkTrigger.trigger`
- Create: `salesforce/force-app/main/default/classes/ContentDocumentLinkTriggerHandler.cls`
- Create: `salesforce/force-app/main/default/classes/ContentDocumentLinkTriggerHandlerTest.cls`

**Step 1:** Implement After Insert trigger: if `LinkedEntityId` references `Messenger_Message__c` or `Messenger_Attachment__c`, set `Has_File__c = true` and increment `File_Count__c`
**Step 2:** Implement After Delete: decrement `File_Count__c`, set `Has_File__c = false` when count reaches 0
**Step 3:** Bulkify: collect all affected message IDs, query once, update once
**Step 4:** Test with bulk scenario (200 ContentDocumentLinks in one transaction)
**Step 5:** Deploy: `sf apex run test --synchronous --class-names ContentDocumentLinkTriggerHandlerTest`
**Step 6:** Commit: `feat: contentdocumentlink trigger for file count denormalization`

---

### Task 4.3: Inbound Media — Go Downloads from Telegram, Uploads ContentVersion via REST API

**Files:**
- Create: `internal/media/inbound.go`
- Create: `internal/media/inbound_test.go`

**Step 1:** Download media from Telegram (via adapter: MTProto getFile / Bot API file download)
**Step 2:** Create `Messenger_Attachment__c` record via REST API
**Step 3:** Upload file to ContentVersion via multipart (`FirstPublishLocationId` = attachment record)
**Step 4:** Update `Messenger_Attachment__c` with `Content_Version_Id__c`, `Content_Document_Id__c`
**Step 5:** Test end-to-end with mocks
**Step 6:** Commit: `feat: inbound media pipeline telegram to contentversion`

---

### Task 4.4: Outbound Media — LWC File Upload to ContentVersion, Go Queries and Sends to Telegram

**Files:**
- Modify: `salesforce/force-app/main/default/lwc/messengerChat/`
- Create: `internal/media/outbound.go`
- Create: `internal/media/outbound_test.go`

**Step 1:** LWC: add `lightning-file-upload` component to chat compose area, creating ContentVersion on upload
**Step 2:** LWC: `onuploadfinished` handler creates `Messenger_Attachment__c` and notifies Go middleware (via Apex Queueable callout)
**Step 3:** Go: receive notification with `ContentVersionId` + Telegram recipient
**Step 4:** Go: download file via ContentVersion `VersionData` endpoint (streaming, no full file buffering)
**Step 5:** Go: send file to Telegram via adapter (MTProto `messages.sendMedia` or Bot API `sendDocument`/`sendPhoto`)
**Step 6:** Test with mock SF download + mock Telegram upload
**Step 7:** Commit: `feat: outbound media pipeline contentversion to telegram`

---

### Task 4.5: LWC Media Rendering via `/sfc/servlet.shepherd/` URLs

**Files:**
- Modify: `salesforce/force-app/main/default/lwc/messengerChat/`
- Create: `salesforce/force-app/main/default/lwc/messengerMediaViewer/`

**Step 1:** Add media display to chat component (images, video, documents)
**Step 2:** Construct servlet URLs: `/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB720BY480&versionId={Content_Version_Id__c}`
**Step 3:** Render `<img>` tags with lazy loading (`loading="lazy"`) for images; type-based rendering for video, voice, documents, stickers
**Step 4:** No CSP Trusted Sites needed (native Salesforce URLs)
**Step 5:** Deploy and verify media renders in scratch org
**Step 6:** Commit: `feat: lwc media rendering with contentversion servlet urls`

---

**Phase 4 complete → notify user with summary**

---

## Phase 5: Real-time Chat

**Agent:** `go-backend` (Platform Event publishing), `salesforce-dev` (empApi + LWC)
**Skills:** `salesforce-apex-patterns`, `data-ingestion`
**Depends on:** Phase 3

### Task 5.1: Platform Event Publisher (Go side)

**Files:**
- Modify: `internal/salesforce/ingestion.go`
- Create: `internal/salesforce/event_publisher.go`
- Create: `internal/salesforce/event_publisher_test.go`

**Step 1:** Implement Platform Event publishing from Go middleware via Apex REST or Composite API
**Step 2:** Publish `Inbound_Message__e` for new messages (already partially implemented in ingester)
**Step 3:** Publish `Session_Status__e` for connection status changes
**Step 4:** Publish `Message_Delivery_Status__e` for delivery confirmations
**Step 5:** Test with mock Salesforce REST API
**Step 6:** Commit: `feat: platform event publisher for real-time updates`

---

### Task 5.2: empApi Subscription in LWC

**Files:**
- Modify: `salesforce/force-app/main/default/lwc/messengerChat/`
- Create: `salesforce/force-app/main/default/lwc/messengerLiveChat/`

**Step 1:** Import `lightning/empApi` in LWC
**Step 2:** Subscribe to `/event/tgint__Inbound_Message__e` for new messages
**Step 3:** Subscribe to `/event/tgint__Message_Delivery_Status__e` for delivery status
**Step 4:** Subscribe to `/event/tgint__Session_Status__e` for connection status
**Step 5:** Display real-time messages without page refresh
**Step 6:** No CSP Trusted Sites needed (empApi is native Salesforce)
**Step 7:** No external auth/JWT needed (empApi uses the user's Salesforce session)
**Step 8:** Deploy and verify live updates in scratch org
**Step 9:** Commit: `feat: lwc real-time chat with empapi platform events`

---

### Task 5.3: empApi Error Handling and Reconnection

**Files:**
- Modify: `salesforce/force-app/main/default/lwc/messengerChat/`
- Modify: `salesforce/force-app/main/default/lwc/messengerLiveChat/`

**Step 1:** Implement `onError` handler for empApi subscription failures
**Step 2:** Implement automatic resubscription on disconnect using `setDebugFlag(true)` for diagnostics
**Step 3:** Handle `-1` replayId for latest events and `-2` for all retained events
**Step 4:** Add UI feedback for connection state (connected/disconnected/reconnecting)
**Step 5:** Test with simulated disconnect scenarios
**Step 6:** Commit: `feat: empapi error handling and reconnection patterns`

---

### Task 5.4: Wire It All Together

**Files:**
- Modify: `cmd/messenger/main.go`
- Modify: `internal/server/server.go`

**Step 1:** Wire inbound message flow: Telegram → adapter → Platform Event publish + SF record ingestion
**Step 2:** Ensure both paths fire concurrently (errgroup)
**Step 3:** Integration test: send mock Telegram update → verify Platform Event publish + SF record creation
**Step 4:** Commit: `feat: end-to-end inbound message pipeline`

---

**Phase 5 complete → notify user with summary**

---

## Phase 6: Polish & Security

**Agent:** `go-backend`, `salesforce-dev`, `security-reviewer`
**Skills:** `outbound-failure-sync`, `rate-limiting`
**Depends on:** Phases 1-5

### Task 6.1: Outbound Message Sending

**Files:**
- Create: `internal/telegram/outbound.go`
- Create: `internal/telegram/outbound_test.go`
- Create: `salesforce/force-app/main/default/classes/MessengerOutboundController.cls`
- Create: `salesforce/force-app/main/default/classes/MessengerOutboundControllerTest.cls`

**Step 1:** Apex: `@AuraEnabled` method to send message (calls Go middleware)
**Step 2:** Go: receive outbound request, route to Telegram via adapter
**Step 3:** Store in outbound_queue (PostgreSQL) for retry
**Step 4:** Test both sides
**Step 5:** Commit: `feat: outbound message sending sf to telegram`

---

### Task 6.2: Delivery Status Sync

**Files:**
- Create: `internal/salesforce/delivery_status.go`
- Create: `internal/salesforce/delivery_status_test.go`
- Create: `salesforce/force-app/main/default/triggers/DeliveryStatusTrigger.trigger`
- Create: `salesforce/force-app/main/default/classes/DeliveryStatusHandler.cls`

**Skill:** `outbound-failure-sync` — MUST invoke

**Step 1:** Go: detect delivery failures (USER_BANNED, PEER_ID_INVALID, etc.)
**Step 2:** Go: publish Message_Delivery_Status__e Platform Event
**Step 3:** Apex trigger: update Messenger_Message__c status
**Step 4:** LWC: empApi subscription for live status updates
**Step 5:** Test terminal vs transient failures
**Step 6:** Commit: `feat: delivery status sync with platform events`

---

### Task 6.3: Bot API Throttling Queue

**Files:**
- Create: `internal/telegram/botapi/throttle.go`
- Create: `internal/telegram/botapi/throttle_test.go`

**Skill:** `rate-limiting` (Bot API section)

**Step 1:** Per-chat rate limiter (1 msg/sec DM, 20 msg/min group)
**Step 2:** Global rate limiter (30 msg/sec)
**Step 3:** Queue with bounded workers via errgroup
**Step 4:** Test rate limiting behavior
**Step 5:** Commit: `feat: bot api throttling with per-chat rate limits`

---

### Task 6.4: Security Audit

**Agent:** `security-reviewer`

**Step 1:** Run security-reviewer agent on entire codebase
**Step 2:** Fix all FAIL findings
**Step 3:** Address WARN findings
**Step 4:** DAST scan setup: document OWASP ZAP configuration
**Step 5:** Verify TLS on all outbound connections
**Step 6:** Commit: `fix: security audit findings`

---

### Task 6.5: Permission Sets

**Files:**
- Create: `salesforce/force-app/main/default/permissionsets/Messenger_Admin.permissionset-meta.xml`
- Create: `salesforce/force-app/main/default/permissionsets/Messenger_User.permissionset-meta.xml`

**Step 1:** Admin: full CRUD on all Messenger_* objects + custom metadata access
**Step 2:** User: read + create on Messages, read on Connections/Chats
**Step 3:** Deploy and test permission boundaries
**Step 4:** Commit: `feat: permission sets for managed package`

---

**Phase 6 complete → notify user with summary**

---

## Phase 7: Packaging & Distribution

**Agent:** `salesforce-dev`
**Skills:** `salesforce-apex-patterns`
**Depends on:** Phase 6

### Task 7.1: 2GP Package Configuration

**Step 1:** Verify all metadata has correct namespace (tgint__)
**Step 2:** Create package version: `sf package version create`
**Step 3:** Install in test org and verify
**Step 4:** Commit: `feat: 2gp managed package version`

---

### Task 7.2: Test Coverage

**Step 1:** Run: `sf apex run test --synchronous --code-coverage`
**Step 2:** Verify >= 75% coverage (target 90%+)
**Step 3:** Add missing tests for uncovered paths
**Step 4:** Commit: `test: increase apex test coverage to 90%+`

---

### Task 7.3: AppExchange Prep

**Skill:** Read `MessageForge.Documentation/plans/appexchange-onboarding.md`

**Step 1:** Complete security checklist (MessageForge.Documentation/reference/security-checklist.md)
**Step 2:** Prepare DAST scan report
**Step 3:** Verify Connected App configuration
**Step 4:** Prepare listing materials
**Step 5:** Commit: `docs: appexchange submission preparation`

---

**Phase 7 complete → notify user: MVP DONE**

---

## Dependency Graph

```
Phase 1 (Foundation)
  ├── Phase 2 (MTProto) ──┐
  └── Phase 3 (Salesforce) ├── Phase 5 (Real-time)
        └── Phase 4 (Media) ┘       │
                                     ├── Phase 6 (Polish)
                                     │       │
                                     └───────┴── Phase 7 (Packaging)
```

**Parallelizable:** Phase 2 + Phase 3 can run concurrently (Go + SF are independent until Phase 5 wires them together).

## Agent Assignment Summary

| Phase | Primary Agent | Supporting Agent | Key Skills |
|---|---|---|---|
| 1 | go-backend | — | — |
| 2 | go-backend | — | telegram-auth-flow, rate-limiting |
| 3 | salesforce-dev + go-backend | question-resolver | salesforce-apex-patterns, data-ingestion |
| 4 | go-backend | salesforce-dev | media-pipeline |
| 5 | go-backend + salesforce-dev | — | salesforce-apex-patterns, data-ingestion |
| 6 | go-backend + salesforce-dev | security-reviewer | outbound-failure-sync, rate-limiting |
| 7 | salesforce-dev | — | salesforce-apex-patterns |
