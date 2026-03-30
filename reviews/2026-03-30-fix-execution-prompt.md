# Fix Register Execution Prompt — 2026-03-30

**Source:** [Fix Register](./2026-03-30-fix-register.md) (15 issues, verified against current codebase)
**Prior roadmap:** [Implementation Roadmap](./2026-03-29-implementation-roadmap.md) — Phases 1.5, 2.5, 3.5, 3.6 are CONFIRMED FIXED
**Scope:** All remaining open issues from the March 30 parallel recheck

---

## How to Use

Copy the prompt below into a new Claude Code conversation. It will execute all 15 fixes from the fix register in dependency order, using parallel agents where possible.

---

## The Prompt

```
Implement all fixes from MessageForge.Documentation/reviews/2026-03-30-fix-register.md in tier order: Tier 0 → Tier 1 → Tier 2 → Tier 3 → Coverage.

Rules:
- For each task: invoke /dev-review-loop with the task description (code → test → review → iterate until 9/10)
- Use go-backend agent (background, isolated) for Go tasks, salesforce-dev agent (background, isolated) for Apex/LWC tasks
- Run independent tasks in parallel as background agents (e.g., Go + SF tasks within same tier)
- After each tier completes: run /parallel-check
- Update MessageForge.Documentation/reviews/2026-03-30-fix-register.md status column as fixes are completed (OPEN → FIXED)
- Update MessageForge.Documentation/reviews/2026-03-29-risk-register.md as risks are resolved
- Only notify me at tier boundaries or if genuinely stuck
- Use /save-session after each tier completes

Reference files for each fix:
- Fix details, code locations, fix options: MessageForge.Documentation/reviews/2026-03-30-fix-register.md
- Architecture context: MessageForge.Documentation/architecture/architecture.md
- Risk register: MessageForge.Documentation/reviews/2026-03-29-risk-register.md
- Salesforce data model: MessageForge.Documentation/reference/sf-data-model.md

Execution order:

TIER 0 — Blocks Integration Testing (3 CRITICAL fixes):

  FIX-001 + FIX-002 run in parallel (independent — Go and SF):

  - FIX-001 (go-backend + salesforce-dev, parallel, background):
    /dev-review-loop Fix outbound JSON contract mismatch between Go and Salesforce.
    Go side: In MessageForge.Backend/internal/messenger/outbound_handler.go:16-22, change
    outboundAPIRequest JSON tags to match Apex camelCase output:
      chat_id → chatExternalId
      message_text → text
      (add) messageType
      media_file_id → mediaUrl
      sf_message_id → sfMessageExternalId
      platform → protocol
    Also update the handler logic and any tests that reference the old field names.
    SF side: Verify MessengerOutboundService.cls:7-14 OutboundMessage class fields match.
    Acceptance: Go can deserialize JSON sent by Apex MessengerOutboundService.

  - FIX-002 (salesforce-dev, background):
    /dev-review-loop Fix SessionStatusTriggerHandler.cls field name mismatch.
    In MessageForge.Salesforce/force-app/main/default/classes/SessionStatusTriggerHandler.cls,
    replace all references to Channel_SF_ID__c with Connection_SF_ID__c (lines 19, 24, 25).
    The correct field is defined in Session_Status__e/fields/Connection_SF_ID__c.field-meta.xml.
    Update the corresponding test class SessionStatusTriggerHandlerTest.cls.
    Acceptance: Session status Platform Events are processed, not silently skipped.

  → Wait FIX-001 + FIX-002 → then start FIX-003 (depends on FIX-001):

  - FIX-003 (go-backend, background):
    /dev-review-loop Wire OutboundWorker with a concrete messenger adapter.
    In MessageForge.Backend/internal/messenger/app.go:256-260, the outboundWorker is left nil.
    Steps:
    1. Identify or create an adapter implementing the messengerAdapter interface
       (likely telegram.Adapter or a wrapper around it)
    2. Wire it into NewApp() when constructing OutboundWorker
    3. Add a startup warning log when outbound adapter is unavailable (graceful degradation)
    4. Verify OutboundWorker.Run() actor is registered and polls outbound_queue
    Reference: outbound_worker.go:27-32 for the struct, :35-55 for Run()
    Acceptance: Messages inserted into outbound_queue via POST /api/outbound are
    dequeued and sent via Telegram. Add integration-style test.

  → Wait FIX-003 → /parallel-check → /save-session

---

TIER 1 — Before MVP Launch (4 HIGH fixes):

  FIX-004 + FIX-005/006 + FIX-007 run in parallel (all Go, independent):

  - FIX-004 (go-backend, background):
    /dev-review-loop Wire inbound queue for durability — currently a dead path.
    Problem: Nothing writes to inbound_queue table. Pipeline uses in-memory channel only.
    QueueWorker polls forever and finds nothing.
    In MessageForge.Backend:
    1. In the webhook handler (internal/telegram/botapi/handlers.go), after receiving a
       Telegram update, INSERT into inbound_queue before pushing to the channel.
       This provides persistence — if the pipeline or Salesforce is down, messages survive.
    2. OR: In pipeline.go, on processMessage() failure, INSERT the failed message into
       inbound_queue for retry by QueueWorker.
    3. Document the chosen approach (Option A or B) in a code comment.
    Acceptance: When Pipeline.processMessage() fails, the message ends up in inbound_queue
    and is retried by QueueWorker. Add test for the failure-to-queue path.

  - FIX-005 + FIX-006 (go-backend, background):
    /dev-review-loop Wire encrypted credentials and require EncryptionKey.
    Two related fixes done together:
    FIX-006: In internal/messenger/config/config.go:48, add validate:"required" to EncryptionKey:
      EncryptionKey string `env:"ENCRYPTION_KEY" validate:"required"`
    FIX-005: Wire crypto.Encrypt/crypto.Decrypt (internal/database/crypto.go) into the
    connection repository read/write path:
    1. Create or update the connection repository (find where connections table is queried)
    2. On write: encrypt credentials JSON with crypto.Encrypt(), store in encrypted_credentials
    3. On read: decrypt encrypted_credentials with crypto.Decrypt(), return as credentials
    4. Add migration 004: UPDATE connections SET encrypted_credentials = encrypt(credentials);
       then ALTER TABLE connections DROP COLUMN credentials;
    5. Update all tests to provide ENCRYPTION_KEY in test config
    Acceptance: go test passes. Bot tokens are stored encrypted. Plaintext column is gone.

  - FIX-007 (go-backend, background):
    /dev-review-loop Remove archived/ directory from Go module to fix go vet.
    The archived/ directory at MessageForge.Backend/archived/ contains .go files with
    broken imports that cause go vet ./... to fail.
    Options (pick simplest):
    A) Move archived/ to MessageForge.Documentation/archived-backend/
    B) Delete archived/ entirely (it's preserved in git history)
    C) Add a go.mod inside archived/ to make it a separate module
    Acceptance: go vet ./... passes clean from MessageForge.Backend/.

  → Wait all → /parallel-check → /save-session

---

TIER 2 — Before Production (5 MEDIUM fixes):

  All 5 are independent — run all in parallel:

  - FIX-008 (go-backend, background):
    /dev-review-loop Replace terminal error string matching with typed errors.
    Problem: isTerminalSendError() in outbound_worker.go:213-228 uses strings.Contains()
    against err.Error(). Duplicate list exists in salesforce/delivery_status.go:17-24
    (which also has USER_BANNED_IN_CHANNEL missing from the other list).
    Steps:
    1. Create a TelegramError struct in internal/telegram/errors.go with a Code field
    2. Have the Telegram adapter return TelegramError for API errors
    3. Replace strings.Contains with errors.As matching on TelegramError.Code
    4. Create a single terminalCodes set shared between outbound_worker and delivery_status
    5. Remove the duplicate list from delivery_status.go
    Acceptance: Terminal error detection uses typed errors. Single source of truth.

  - FIX-009 (go-backend, background):
    /dev-review-loop Replace GlobalMetrics singleton with dependency injection.
    Problem: server.GlobalMetrics is a mutable package-level var (internal/server/metrics.go:26).
    Workers in internal/messenger/ import server package to access it — layering violation.
    Steps:
    1. Add *Metrics parameter to NewQueueWorker() and NewOutboundWorker() constructors
    2. Replace server.GlobalMetrics references in queue_worker.go:129,136 and
       outbound_worker.go:146,158 with the injected instance
    3. Remove var GlobalMetrics from metrics.go (or keep for backward compat if needed)
    4. Wire the Metrics instance through app.go NewApp()
    Acceptance: messenger package no longer imports server package. Tests use isolated metrics.

  - FIX-010 (go-backend, background):
    /dev-review-loop Add content-type allowlist to outbound media presign handler.
    Problem: In internal/media/outbound.go:64, req.ContentType is passed directly to
    R2 presigner without validation. Could enable stored XSS via text/html content type.
    Steps:
    1. Define allowedContentTypes map in outbound.go:
       image/jpeg, image/png, image/gif, image/webp, video/mp4,
       audio/ogg, audio/mpeg, application/pdf, application/zip
    2. Validate req.ContentType against the allowlist before calling PresignedPutURL
    3. Return 400 with clear error for disallowed types
    4. Add test for rejected content type
    Acceptance: Only allowlisted content types produce presigned URLs.
    Note: This handler is not yet registered in app.go — fix preemptively before wiring.

  - FIX-011 (salesforce-dev, background):
    /dev-review-loop Resolve Chat_SF_ID__c field naming ambiguity on Inbound_Message__e.
    Problem: InboundMessageTriggerHandler.cls:25-32 uses Chat_SF_ID__c for both Salesforce
    record IDs and external chat IDs (Telegram chat IDs). If an external ID happens to be
    18 alphanumeric characters, it could be misinterpreted as a Salesforce ID.
    Recommended fix (Option C — least disruptive):
    1. Add a prefix convention: Go sends external IDs with "ext:" prefix (e.g., "ext:-1001234567890")
    2. InboundMessageTriggerHandler checks for "ext:" prefix to distinguish
    3. Document the convention in the field help text
    4. Update Go ingester (internal/salesforce/ingestion.go) to add "ext:" prefix
    OR Option A (cleaner but more work):
    1. Add Chat_External_ID__c field to Inbound_Message__e
    2. Go sends external ID in Chat_External_ID__c, SF ID in Chat_SF_ID__c (when known)
    3. Update InboundMessageTriggerHandler to check Chat_External_ID__c first
    Pick the option that best fits the current architecture.
    Acceptance: No ambiguity between SF IDs and external IDs. Test with 18-char external ID.

  - FIX-012 (salesforce-dev, background):
    /dev-review-loop Remove console.error() statements from production LWC code.
    Locations (7 total):
      messengerChat.js: lines 51, 73, 92, 198, 282
      messengerLiveChat.js: lines 28, 56
    Replace with either:
    A) Remove entirely (errors are already handled via toast/UI)
    B) Guard with a debug mode flag
    Do NOT break error handling — ensure the catch blocks still handle errors properly.
    Acceptance: Zero console.error calls in production LWC. AppExchange security scanner clean.

  → Wait all → /parallel-check → /save-session

---

TIER 3 — Code Quality (3 LOW fixes + coverage):

  All independent — run in parallel:

  - FIX-013 (go-backend, background):
    /dev-review-loop Fix transaction rollback using cancelled context.
    In queue_worker.go:83 and outbound_worker.go:65, the deferred Rollback(ctx) runs
    with an already-cancelled context on SIGINT. Fix:
    Change `_ = dbTx.Rollback(ctx)` to use context.Background() with a short timeout:
      rctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
      defer cancel()
      _ = dbTx.Rollback(rctx)
    Acceptance: Rollback uses a valid context. No functional change but correct semantics.

  - FIX-014 (salesforce-dev, background):
    /dev-review-loop Improve DML exception logging in Salesforce trigger handlers.
    Locations:
      InboundMessageTriggerHandler.cls:130
      DeliveryStatusTriggerHandler.cls:89
      SessionStatusTriggerHandler.cls:64
    Change from generic logging to detailed:
      } catch (DmlException e) {
          for (Integer i = 0; i < e.getNumDml(); i++) {
              System.debug(LoggingLevel.ERROR, 'DML Error [' + i + ']: ' +
                  'Fields: ' + e.getDmlFieldNames(i) +
                  ' | Status: ' + e.getDmlStatusCode(i) +
                  ' | Message: ' + e.getDmlMessage(i));
          }
      }
    Acceptance: DML errors include field names and status codes in debug logs.

  - FIX-015 (salesforce-dev, background):
    /dev-review-loop Add bulk test scenarios (200+ records) to Salesforce trigger handlers.
    Create bulk test methods in:
      InboundMessageTriggerHandlerTest.cls
      DeliveryStatusTriggerHandlerTest.cls
      SessionStatusTriggerHandlerTest.cls
    Each test should create 200+ Platform Event records and verify:
    1. No governor limit exceptions
    2. All records processed correctly
    3. Correct DML count (bulkified)
    Acceptance: All bulk tests pass. No governor limit violations.

  - COVERAGE (go-backend, background):
    /dev-review-loop Increase Go test coverage from 50.1% toward 80% target.
    Priority packages (current → target):
      internal/messenger: 16.8% → 60%+ (test app.go NewApp happy path, queue_worker, outbound_worker)
      internal/database: 38.4% → 70%+ (test migrations.go, session_backup.go)
      internal/server: 49.0% → 70%+ (test server.go RegisterWebhook, AdminRun)
      internal/media: 62.9% → 80%+ (test r2.go with mock S3 client)
    Use table-driven tests. Mock external dependencies (pgx pool, S3 client, HTTP).
    Do NOT test main.go (cmd/messenger) — skip it.
    Acceptance: go test -coverprofile shows 70%+ total. Race detector clean.

  → Wait all → /parallel-check → /save-session

---

FINAL:
  - Update MessageForge.Documentation/reviews/2026-03-30-fix-register.md — mark all fixes as FIXED
  - Update MessageForge.Documentation/reviews/2026-03-29-risk-register.md with final status
  - Run final /parallel-check across entire codebase
  - Report summary: fixes applied, coverage before/after, any remaining issues
```

---

## Dependency Graph

```
TIER 0 (CRITICAL)
  FIX-001 (Go+SF: outbound JSON) ──┐
  FIX-002 (SF: field name)     ────┤── parallel
                                   │
                                   ▼
                               FIX-003 (Go: wire OutboundWorker) ── depends on FIX-001
                                   │
                                   ▼
                              /parallel-check

TIER 1 (HIGH)
  FIX-004 (Go: inbound queue) ────┐
  FIX-005+006 (Go: encryption) ───┤── all parallel
  FIX-007 (Go: archived/)     ────┘
                                   │
                                   ▼
                              /parallel-check

TIER 2 (MEDIUM)
  FIX-008 (Go: typed errors)  ────┐
  FIX-009 (Go: metrics DI)    ────┤
  FIX-010 (Go: content-type)  ────┤── all parallel
  FIX-011 (SF: field naming)  ────┤
  FIX-012 (SF: console.error) ────┘
                                   │
                                   ▼
                              /parallel-check

TIER 3 (LOW + COVERAGE)
  FIX-013 (Go: rollback ctx)  ────┐
  FIX-014 (SF: DML logging)  ────┤── all parallel
  FIX-015 (SF: bulk tests)   ────┤
  COVERAGE (Go: 50%→70%+)    ────┘
                                   │
                                   ▼
                              /parallel-check → final report
```

## Estimated Effort

| Tier | Fixes | Agents | Estimated Time |
|---|---|---|---|
| Tier 0 | 3 (FIX-001→003) | go-backend + salesforce-dev | 4-6 hours |
| Tier 1 | 4 (FIX-004→007) | go-backend | 8-12 hours |
| Tier 2 | 5 (FIX-008→012) | go-backend + salesforce-dev | 6-8 hours |
| Tier 3 | 3 + coverage (FIX-013→015) | go-backend + salesforce-dev | 4-8 hours |
| **Total** | **15 + coverage** | | **22-34 hours** |

## Verification Commands (per tier)

```bash
# After every tier:
cd MessageForge.Backend && go vet ./... && go test -race ./... -count=1

# After Tier 1 (encryption):
cd MessageForge.Backend && go test ./internal/database/... -v -run TestEncrypt

# After Tier 2 (SF changes):
cd MessageForge.Salesforce && sf apex run test --test-level RunLocalTests --wait 30

# After Tier 3 (coverage):
cd MessageForge.Backend && go test -coverprofile=coverage.out ./... && go tool cover -func=coverage.out | tail -1

# Final:
cd MessageForge.Backend && govulncheck ./...
```
