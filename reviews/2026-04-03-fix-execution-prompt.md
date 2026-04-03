# MessageForge Fix Execution Prompt — 2026-04-03

> Copy-paste this prompt into a new Claude Code session to execute all fixes autonomously.

---

```
You are executing a comprehensive fix plan for the MessageForge project based on the verification report at:
MessageForge.Documentation/reviews/2026-04-03-full-verification-report.md

## Ground Rules

1. **Autonomous execution** — use /dev-review-loop skill for each workstream. Do NOT ask the user. Use question-resolver agent for decisions.
2. **Parallel workstreams** — launch independent workstreams as parallel agents. Merge results at the end.
3. **Self-review loop** — each workstream: implement → test → review → fix → repeat until 9/10. Use code-reviewer and security-reviewer agents.
4. **Auto-deploy SF** — after each Salesforce change, deploy to sandbox and run tests:
   ```bash
   cd MessageForge.Salesforce && sf project deploy start --source-dir force-app --target-org sandbox --wait 10
   sf apex run test --target-org sandbox --synchronous --code-coverage --result-format human
   ```
5. **Go verification** — after each Go change:
   ```bash
   cd MessageForge.Backend && go vet ./... && go test ./... -race -count=1
   ```
6. **Commit after each workstream** — one commit per completed workstream. Conventional commits format.
7. **No regressions** — all 15 prior fixes (FIX-001 through FIX-015) must remain intact. Run full test suites before committing.

## Repositories

- Go backend: MessageForge.Backend/
- Salesforce: MessageForge.Salesforce/
- Documentation: MessageForge.Documentation/
- SF sandbox org alias: `sandbox`

## Read First (mandatory context)

Before starting ANY work, read these files:
- MessageForge.Documentation/reviews/2026-04-03-full-verification-report.md (the verification report driving all fixes)
- MessageForge.Documentation/architecture/adr.md (ADR-20, ADR-21 especially)
- MessageForge.Documentation/reference/postgres-schema.md
- MessageForge.Documentation/reference/sf-data-model.md
- MessageForge.Documentation/reference/security-checklist.md
- MessageForge.Documentation/reference/deployment-guide.md
- .claude/skills/media-pipeline.md (ContentVersion implementation guide)
- .claude/skills/salesforce-apex-patterns.md (FLS/CRUD patterns)

---

## WORKSTREAM A: Centrifugo Removal (ADR-21) [Go + SF]

**Priority: CRITICAL | Parallel: YES — independent of other workstreams**

Remove ALL Centrifugo code per ADR-21. This is pure deletion with no new features.

### Go Backend — delete these files/sections:
1. Delete `MessageForge.Backend/internal/realtime/centrifugo.go`
2. Delete `MessageForge.Backend/internal/realtime/centrifugo_test.go`
3. If `internal/realtime/` is now empty, delete the entire directory
4. Remove Centrifugo config fields from `MessageForge.Backend/internal/messenger/config/config.go`:
   - `CentrifugoURL`, `CentrifugoAPIKey`, `CentrifugoSecret` and their env tags
5. Remove Centrifugo service from `MessageForge.Backend/docker-compose.yml` (lines 19-37 approximately — the `centrifugo:` service block)
6. In `MessageForge.Backend/internal/messenger/app.go` — remove any `realtimePublisher` or Centrifugo wiring. If `pipeline.go` references a `realtimePublisher` interface, check if it's still used for anything else. If only Centrifugo used it, remove the interface and the field.
7. Check `MessageForge.Backend/internal/messenger/pipeline.go` for Centrifugo references — clean up.

### Salesforce — delete these files:
1. Delete `MessageForge.Salesforce/force-app/main/default/classes/CentrifugoTokenController.cls`
2. Delete `MessageForge.Salesforce/force-app/main/default/classes/CentrifugoTokenController.cls-meta.xml`
3. Delete `MessageForge.Salesforce/force-app/main/default/classes/CentrifugoTokenControllerTest.cls`
4. Delete `MessageForge.Salesforce/force-app/main/default/classes/CentrifugoTokenControllerTest.cls-meta.xml`
5. Delete entire `MessageForge.Salesforce/force-app/main/default/lwc/centrifugoClient/` directory
6. Delete entire `MessageForge.Salesforce/force-app/main/default/lwc/messengerLiveChat/` directory
7. Remove Centrifugo fields from Custom Metadata Types:
   - `MessageForge.Salesforce/force-app/main/default/objects/Middleware_Config__mdt/fields/Centrifugo_URL__c.field-meta.xml` — delete file
   - `MessageForge.Salesforce/force-app/main/default/objects/Middleware_Config__mdt/fields/Centrifugo_HMAC_Secret__c.field-meta.xml` — delete file
   - `MessageForge.Salesforce/force-app/main/default/objects/Messenger_Settings__mdt/fields/Centrifugo_URL__c.field-meta.xml` — delete file
8. In `messengerChat.js` — remove the comment about Centrifugo (line ~162) if present. Do NOT break the existing empApi functionality.

### Stale references to clean:
- `MessageForge.Salesforce/.claude/skills/centrifugo-integration.md` — delete file
- Any Centrifugo references in `.claude/agents/question-resolver.md` — edit out
- Delete `MessageForge.Centrifugo/` directory entirely (it's just a placeholder CLAUDE.md)

### Verification:
```bash
cd MessageForge.Backend && grep -r -i "centrifugo" --include="*.go" internal/ cmd/
cd MessageForge.Backend && go vet ./... && go test ./... -race -count=1
cd MessageForge.Salesforce && grep -r -i "centrifugo" --include="*.cls" --include="*.js" --include="*.xml" force-app/
cd MessageForge.Salesforce && sf project deploy start --source-dir force-app --target-org sandbox --wait 10
sf apex run test --target-org sandbox --synchronous --code-coverage
```

Zero Centrifugo references should remain in source code (skills/docs references are OK if marked deprecated).

---

## WORKSTREAM B: R2 Removal + ContentVersion Implementation (ADR-20) [Go]

**Priority: CRITICAL | Parallel: YES — independent of Workstream A**

Two phases: (1) remove R2 dead code, (2) implement ContentVersion client.

### Phase B1: R2 Removal

Delete these files entirely:
1. `MessageForge.Backend/internal/media/r2.go`
2. `MessageForge.Backend/internal/media/r2_test.go`
3. `MessageForge.Backend/internal/media/r2_mock_test.go`
4. `MessageForge.Backend/workers/media-proxy/` — entire directory (Cloudflare Worker)

Remove R2 config fields from `MessageForge.Backend/internal/messenger/config/config.go`:
- `R2AccountID`, `R2AccessKeyID`, `R2SecretAccessKey`, `R2BucketName`

Fix stale references:
- `MessageForge.Backend/internal/media/inbound.go` — update R2 comments to ContentVersion
- `MessageForge.Backend/internal/media/outbound.go` — update R2 references to ContentVersion
- `MessageForge.Backend/internal/media/inbound_test.go` — update R2 test URLs
- `MessageForge.Backend/internal/media/outbound_test.go` — update R2 test URLs
- `MessageForge.Backend/internal/common/models/models.go:32` — change `// R2 URL after media processing` to `// ContentVersion URL after media processing`

Create migration `005_drop_r2_columns.sql`:
```sql
-- Migration 005: Drop R2 columns and media_files table (ADR-20)
DROP TABLE IF EXISTS media_files;
ALTER TABLE connections DROP COLUMN IF EXISTS credentials;
INSERT INTO schema_migrations (version) VALUES (5);
```
Note: This also drops the legacy plaintext `credentials` column (Security Finding #5).

Fix SF stale references:
- `MessageForge.Salesforce/force-app/main/default/objects/Messenger_Attachment__c/Messenger_Attachment__c.object-meta.xml` — change description from "Files stored in R2" to "Files stored in Salesforce ContentVersion"
- `MessageForge.Salesforce/force-app/main/default/objects/Middleware_Config__mdt/fields/Media_CDN_URL__c.field-meta.xml` — update description from "Cloudflare Worker" reference
- `MessageForge.Salesforce/.claude/skills/question-resolver.md:53` — delete the rule "Media in R2 -- never store media in Salesforce" — it directly contradicts ADR-20

### Phase B2: ContentVersion Go Client

Read `.claude/skills/media-pipeline.md` for the full implementation guide.

Create `MessageForge.Backend/internal/salesforce/contentversion.go`:
- `UploadContentVersion(ctx, title, filename string, data []byte) (contentDocumentID string, err error)` — multipart upload
- `DownloadContentVersion(ctx, contentDocumentID string) (data []byte, mimeType string, err error)` — download via VersionData URL
- `CreateContentDocumentLink(ctx, contentDocumentID, linkedEntityID string) error` — link to Message record

Use TDD: write `contentversion_test.go` first with table-driven tests, then implement.

Update `internal/media/inbound.go`:
- Replace R2 upload call with ContentVersion upload
- Store ContentDocumentId instead of R2 key

Update `internal/media/outbound.go`:
- Replace R2 presigned URL generation with ContentVersion download
- Remove presigned URL logic entirely

### Verification:
```bash
cd MessageForge.Backend && grep -r "r2\|R2\|cloudflare\|Cloudflare" --include="*.go" internal/ cmd/ workers/ 2>/dev/null
cd MessageForge.Backend && go vet ./... && go test ./... -race -count=1 -cover
```

Zero R2/Cloudflare references should remain.

---

## WORKSTREAM C: Apex Security Fixes (FLS/CRUD + Async Callouts) [SF]

**Priority: CRITICAL + HIGH | Parallel: YES — independent of A and B**

### C1: FLS/CRUD Checks in ChannelSetupController.cls (CRITICAL)

Read the file first: `MessageForge.Salesforce/force-app/main/default/classes/ChannelSetupController.cls`

Fix all DML operations to use `Security.stripInaccessible()`:

1. **`createChannel` method (~line 65)**: Before `insert` on `Messenger_Channel__c`, wrap with:
   ```apex
   SObjectAccessDecision decision = Security.stripInaccessible(AccessType.CREATABLE, new List<SObject>{channel});
   insert decision.getRecords();
   ```

2. **`createChannelWithConfig` method (~line 191)**: Same pattern for both channel and config object inserts.

3. **`createPlatformConfig` helper (~line 340)**: The dynamic SObject built from `credentialsJson` — validate each field with `fieldDescribe.isCreateable()` before `put()`:
   ```apex
   Map<String, Schema.SObjectField> fieldMap = configType.getDescribe().fields.getMap();
   for (String fieldName : credentialFields.keySet()) {
       Schema.DescribeFieldResult fieldDesc = fieldMap.get(fieldName)?.getDescribe();
       if (fieldDesc != null && fieldDesc.isCreateable()) {
           configObj.put(fieldName, credentialFields.get(fieldName));
       }
   }
   ```

4. **`activateChannel` method (~line 101)**: Before `update`, wrap with `Security.stripInaccessible(AccessType.UPDATABLE, ...)`.

5. **`archiveChannel` method (~line 120)**: Same as activateChannel.

6. **Channel_Audit_Log__c inserts**: Any audit log creation should also use `stripInaccessible`.

Also add FLS checks to any other class found doing DML without FLS. Grep for `insert ` and `update ` across all .cls files.

### C2: Async Outbound Callouts (HIGH)

Read `MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls`

The `sendMessage` method (~line 116) makes a synchronous callout. Refactor to use Queueable:

1. Create `MessengerOutboundQueueable.cls`:
   ```apex
   public class MessengerOutboundQueueable implements Queueable, Database.AllowsCallouts {
       private final String messageJson;

       public MessengerOutboundQueueable(String messageJson) {
           this.messageJson = messageJson;
       }

       public void execute(QueueableContext context) {
           MessengerOutboundService.sendMessage(messageJson);
       }
   }
   ```

2. Update `MessengerController.sendMessage` to enqueue instead of direct callout:
   ```apex
   System.enqueueJob(new MessengerOutboundQueueable(JSON.serialize(msg)));
   ```

3. Write `MessengerOutboundQueueableTest.cls` with Test.startTest/stopTest pattern.

### Verification:
```bash
cd MessageForge.Salesforce && sf project deploy start --source-dir force-app --target-org sandbox --wait 10
sf apex run test --target-org sandbox --synchronous --code-coverage --result-format human
```

All tests must pass. Verify no `without sharing` regressions.

---

## WORKSTREAM D: Go Quick Fixes (HIGH) [Go]

**Priority: HIGH | Parallel: YES — after A and B complete, or independent if careful**

### D1: Fix namespace prefix in ingestion.go

Read `MessageForge.Backend/internal/salesforce/ingestion.go`

Find where the Platform Event name `Inbound_Message__e` is used and add the `tgint__` namespace prefix:
- Should be `tgint__Inbound_Message__e` to match the managed package namespace
- Check how `delivery_status.go` does it (it correctly uses `tgint__Message_Delivery_Status__e`) and follow the same pattern

### D2: Fix client_test.go field name

Read `MessageForge.Backend/internal/salesforce/client_test.go`

Find the reference to `External_ID__c` on `Inbound_Message__e` (~line 88) and fix to the correct field name. Check actual `Inbound_Message__e` fields to determine the right one (likely `Chat_External_ID__c` or `Message_External_ID__c`).

### D3: Add HTTP rate limiting middleware

Add rate limiting to the public HTTP server in `MessageForge.Backend/internal/server/`:

1. Add dependency: `go get golang.org/x/time/rate`
2. Create `ratelimit.go` in `internal/server/`:
   ```go
   func RateLimit(limit rate.Limit, burst int) func(http.Handler) http.Handler {
       limiter := rate.NewLimiter(limit, burst)
       return func(next http.Handler) http.Handler {
           return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
               if !limiter.Allow() {
                   http.Error(w, "too many requests", http.StatusTooManyRequests)
                   return
               }
               next.ServeHTTP(w, r)
           })
       }
   }
   ```
3. Apply to the public mux in `server.go` (not the admin mux).
4. Write `ratelimit_test.go` with table-driven tests.

### D4: Add explicit TLS config to auth.go

In `MessageForge.Backend/internal/salesforce/auth.go` (~line 178), add explicit TLS configuration:
```go
transport := &http.Transport{
    TLSClientConfig: &tls.Config{
        MinVersion: tls.VersionTLS12,
    },
}
client := &http.Client{Transport: transport}
```

### D5: Implement sync.Pool in ingestion.go (ADR-17)

Read `MessageForge.Backend/internal/salesforce/ingestion.go`

Add `sync.Pool` for buffer reuse in the batch JSON encoder:
```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}
```
Use in the flush method. Zero the buffer after use: `buf.Reset()`.

Or, if sync.Pool is not appropriate here, update ADR-17 to "deferred" and remove the false claim from `security-checklist.md`.

### Verification:
```bash
cd MessageForge.Backend && go vet ./... && go test ./... -race -count=1 -cover
```

---

## WORKSTREAM E: Test Coverage Boost [Go]

**Priority: HIGH | Parallel: YES — after A and B to avoid testing dead code**

Target: raise all critical packages above 80%.

### Packages needing coverage:
1. `internal/database` (49% → 80%+) — add tests for pool edge cases, migration error paths, connection CRUD operations, session backup edge cases
2. `internal/messenger` (63% → 80%+) — add tests for outbound_store, queue_worker error paths, pipeline error handling, app initialization
3. `internal/salesforce` (79.9% → 80%+) — small push needed, add edge case tests
4. `internal/telegram/botapi` (78.8% → 80%+) — add handler edge cases
5. `internal/telegram/mtproto` (77.4% → 80%+) — add client error paths (lower priority if not MVP)

Use TDD agent for each package. Write table-driven tests covering:
- Happy path (already mostly covered)
- Error paths (database errors, network errors, invalid input)
- Edge cases (empty batches, max sizes, concurrent access)
- Context cancellation handling

### Verification:
```bash
cd MessageForge.Backend && go test -cover ./... 2>&1 | grep -E "coverage:|FAIL"
```

All critical packages must show 80%+ coverage. No FAIL results.

---

## WORKSTREAM F: Documentation Updates [Docs]

**Priority: MEDIUM | Parallel: YES — independent of code changes**

### F1: Update sf-data-model.md
File: `MessageForge.Documentation/reference/sf-data-model.md`
- Fix `Session_Status__e.Channel_SF_ID__c` → `Connection_SF_ID__c`
- Add missing `Inbound_Message__e` fields: `Chat_SF_ID__c`, `Chat_External_ID__c`, `Media_MIME_Type__c`, `Protocol__c`
- Add undocumented objects: `Messenger_Event__c`, `Messenger_Settings__mdt`, `Middleware_Config__mdt`
- Document `Messenger_Message__c.Last_Error__c`

### F2: Update postgres-schema.md
File: `MessageForge.Documentation/reference/postgres-schema.md`
- Add `encrypted_credentials BYTEA NOT NULL` to `connections` table
- Document `UNIQUE` and `ON DELETE CASCADE` constraints on `mtproto_sessions`
- Update `media_files` status: "Dropped in migration 005" (after Workstream B)
- Document `credentials` column drop (after Workstream B)

### F3: Update deployment-guide.md
File: `MessageForge.Documentation/reference/deployment-guide.md`
- Add missing env vars: `TELEGRAM_BOT_TOKEN`, `ADMIN_HOST`, `ADMIN_PORT`
- Remove R2 and Centrifugo env var references
- Document admin/metrics server on port 9090
- Update firewall rules to include port 9090
- Update docker-compose description (PostgreSQL only, no Centrifugo)

### F4: Update security-checklist.md
File: `MessageForge.Documentation/reference/security-checklist.md`
- Remove or correct `sync.Pool buffers zeroed` claim (based on D5 outcome)
- Update media references from R2 to ContentVersion
- Mark FLS/CRUD as implemented (after Workstream C)

### F5: Clean stale docs
- Delete `MessageForge.Backend/draftdocs/CLAUDE.md`
- Update `MessageForge.Backend/docs/wiki/data-model-and-relationships.md` — replace all `sf_connection_id` → `sf_channel_id`, `Messenger_Connection__c` → `Messenger_Channel__c`

### F6: Clarify dual ingestion paths
Add a section to `MessageForge.Documentation/architecture/architecture.md` explaining:
- Go publishes Platform Events directly via Salesforce REST API (primary path)
- `MessengerInboundAPI.cls` REST endpoint exists for batch HTTP ingestion (secondary/alternative path)
- Document when each is used

### Verification:
Review each changed doc for internal consistency. No code changes in this workstream.

---

## Execution Order

```
Phase 1 (parallel):
  ├── WORKSTREAM A: Centrifugo Removal (Go agent + SF deploy)
  ├── WORKSTREAM B: R2 Removal + ContentVersion (Go agent)
  ├── WORKSTREAM C: Apex Security Fixes (SF agent + deploy)
  └── WORKSTREAM F: Documentation Updates (docs only)

Phase 2 (after A+B complete):
  ├── WORKSTREAM D: Go Quick Fixes
  └── WORKSTREAM E: Test Coverage Boost

Phase 3 (final):
  └── Full regression test (Go + SF)
  └── Final commit + summary
```

## Commit Strategy

One commit per workstream:
1. `chore: remove Centrifugo code per ADR-21`
2. `feat: replace R2 with ContentVersion media pipeline per ADR-20`
3. `fix(security): add FLS/CRUD checks and async callouts in Apex`
4. `fix: namespace prefix, rate limiting, TLS config, sync.Pool`
5. `test: raise coverage above 80% on critical packages`
6. `docs: update stale documentation to match current codebase`

## Completion Criteria

Before declaring done:
- [ ] Zero Centrifugo references in source code
- [ ] Zero R2/Cloudflare references in source code
- [ ] ContentVersion upload/download working in Go
- [ ] All Apex DML operations have FLS checks
- [ ] All Go tests pass with `go vet` clean
- [ ] All SF tests pass on sandbox
- [ ] All critical Go packages at 80%+ coverage
- [ ] All documentation matches current code
- [ ] No regressions on FIX-001 through FIX-015

Notify user ONLY at completion with a summary table of what was done.
```
