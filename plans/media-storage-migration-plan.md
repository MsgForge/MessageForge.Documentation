> **DEPRECATED (ADR-20, 2026-03-30):** This migration plan described moving from R2 to ContentVersion. The migration is now the active architecture. See architecture.md for current state.

# Media Storage Migration Plan: R2 to Salesforce ContentVersion

**Date:** 2026-03-30
**ADR:** [ADR-20](../architecture/adr-20-media-storage-pivot.md) (Accepted)
**Scope:** Replace Cloudflare R2 media pipeline with Salesforce ContentVersion across Go middleware, Apex, and LWC.

---

## Phase Overview

```
Phase 1: Foundation (data model + Go adapter)
    │
Phase 2: Inbound Flow (Telegram -> SF Files)
    │
Phase 3: LWC Display (show files in chat)
    │
Phase 4: Outbound Flow (SF -> Telegram)
    │
Phase 5: Cleanup & Polish
```

Phases are sequential. Each phase produces a testable deliverable. Phase 1 and Phase 2 are the critical path for MVP media support.

---

## Phase 1: Foundation (Data Model + Go Adapter)

**Agent:** `go-backend` (adapter), `salesforce-dev` (data model)
**Dependencies:** Phase 1 of MVP plan (Go module, config, database, HTTP server)
**Verification:** `go test ./internal/salesforce/... -v` and `sf project deploy start`

### Task 1.1: Add New Fields to Salesforce Objects

**Agent:** `salesforce-dev`
**Files:**
- Modify: `Messenger_Message__c` object metadata
- Modify: `Messenger_Attachment__c` object metadata

**Changes:**

| Object | Field | Type | Purpose |
|---|---|---|---|
| `Messenger_Message__c` | `Has_File__c` | Checkbox (default: false) | Denormalized flag for query filtering |
| `Messenger_Message__c` | `File_Count__c` | Number(3,0) (default: 0) | Count of attached files |
| `Messenger_Attachment__c` | `Content_Version_Id__c` | Text(18) | Reference to ContentVersion record |
| `Messenger_Attachment__c` | `Content_Document_Id__c` | Text(18) | Reference to ContentDocument record |
| `Messenger_Attachment__c` | `Thumbnail_Content_Version_Id__c` | Text(18) | Video thumbnail ContentVersion |
| `Messenger_Attachment__c` | `File_Size_Bytes__c` | Number(10,0) | File size for display and monitoring |
| `Messenger_Attachment__c` | `Content_Type__c` | Text(100) | MIME type for rendering logic |

**Steps:**
1. Add field metadata XML for each field
2. Deploy to scratch org: `sf project deploy start --source-dir force-app`
3. Verify fields exist: `sf data query --query "SELECT Id FROM Messenger_Message__c LIMIT 1"`
4. Commit: `feat: add ContentVersion fields to message and attachment objects`

---

### Task 1.2: Build Go ContentVersion Upload Adapter

**Agent:** `go-backend`
**Files:**
- Create: `internal/salesforce/contentversion.go`
- Create: `internal/salesforce/contentversion_test.go`

**Interface:**

```go
type ContentVersionUploader interface {
    Upload(ctx context.Context, params UploadParams) (ContentVersionResult, error)
}

type UploadParams struct {
    Title       string    // "photo_2026-03-30.jpg"
    PathOnClient string   // "photo_2026-03-30.jpg"
    ContentType  string   // "image/jpeg"
    Body         io.Reader // file binary stream
    LinkedEntityID string // Messenger_Message__c or Messenger_Attachment__c record ID
}

type ContentVersionResult struct {
    ContentVersionID  string // 068xxxxx
    ContentDocumentID string // 069xxxxx
}
```

**Implementation:**
1. Multipart form-data POST to `/services/data/v62.0/sobjects/ContentVersion`
2. Set `FirstPublishLocationId` to `LinkedEntityID` (auto-creates ContentDocumentLink)
3. Query back `ContentDocumentId` from the created ContentVersion
4. Return both IDs for storage on `Messenger_Attachment__c`

**Steps:**
1. Write failing test with mock HTTP server returning ContentVersion creation response
2. Implement multipart upload using `mime/multipart` writer
3. Implement ContentDocumentId query-back
4. Run: `go test ./internal/salesforce/ -v -run TestContentVersion`
5. Commit: `feat: contentversion multipart upload adapter`

---

### Task 1.3: Build Go ContentVersion Download Adapter

**Agent:** `go-backend`
**Files:**
- Modify: `internal/salesforce/contentversion.go`
- Modify: `internal/salesforce/contentversion_test.go`

**Interface:**

```go
type ContentVersionDownloader interface {
    Download(ctx context.Context, contentVersionID string) (io.ReadCloser, string, error)
    // Returns: body stream, content-type, error
}
```

**Implementation:**
1. GET `/services/data/v62.0/sobjects/ContentVersion/{id}/VersionData`
2. Stream response body (do not buffer entire file in memory)
3. Return `io.ReadCloser` for caller to consume and close

**Steps:**
1. Write failing test with mock HTTP server returning binary data
2. Implement streaming download
3. Run: `go test ./internal/salesforce/ -v -run TestContentVersionDownload`
4. Commit: `feat: contentversion streaming download adapter`

---

### Task 1.4: Build Media Upload Queue with Rate Limiting

**Agent:** `go-backend`
**Files:**
- Create: `internal/media/queue.go`
- Create: `internal/media/queue_test.go`

**Implementation:**
1. Bounded channel-based queue (configurable capacity, default: 1000)
2. Worker pool (configurable concurrency, default: 5)
3. Rate limiter using `golang.org/x/time/rate` (default: 50 uploads/minute)
4. API budget check before each upload via `/services/data/v62.0/limits/`
5. Backpressure: when budget < 20% remaining, pause uploads and retry with exponential backoff

**Steps:**
1. Write failing test for queue capacity and rate limiting behavior
2. Implement queue with worker pool
3. Implement budget check integration
4. Run: `go test ./internal/media/ -v`
5. Commit: `feat: media upload queue with api budget rate limiting`

---

## Phase 2: Inbound Flow (Telegram -> SF Files)

**Agent:** `go-backend`
**Dependencies:** Phase 1 (adapter + queue), MVP Phase 2 (MTProto client)
**Verification:** `go test ./internal/media/... -v` and integration test with scratch org

### Task 2.1: Wire Inbound Pipeline to ContentVersion Upload

**Agent:** `go-backend`
**Files:**
- Modify: `internal/media/inbound.go` (or create if not yet exists)
- Modify: `internal/media/inbound_test.go`

**Flow:**
```
Telegram message with media
    -> Go downloads file from Telegram (MTProto getFile / Bot API file download)
    -> Go creates Messenger_Attachment__c record via REST API
    -> Go uploads file to ContentVersion via multipart (FirstPublishLocationId = attachment record)
    -> Go updates Messenger_Attachment__c with Content_Version_Id__c, Content_Document_Id__c
```

**Steps:**
1. Write failing test with mock Telegram file download + mock SF REST API
2. Implement download-from-Telegram + upload-to-ContentVersion pipeline
3. Handle video thumbnail extraction (ffmpeg first frame -> separate ContentVersion upload)
4. Run: `go test ./internal/media/ -v -run TestInbound`
5. Commit: `feat: inbound media pipeline telegram to contentversion`

---

### Task 2.2: ContentDocumentLink Trigger for Has_File__c

**Agent:** `salesforce-dev`
**Files:**
- Create: `force-app/main/default/triggers/ContentDocumentLinkTrigger.trigger`
- Create: `force-app/main/default/triggers/ContentDocumentLinkTrigger.trigger-meta.xml`
- Create: `force-app/main/default/classes/ContentDocumentLinkTriggerHandler.cls`
- Create: `force-app/main/default/classes/ContentDocumentLinkTriggerHandler.cls-meta.xml`
- Create: `force-app/main/default/classes/ContentDocumentLinkTriggerHandlerTest.cls`
- Create: `force-app/main/default/classes/ContentDocumentLinkTriggerHandlerTest.cls-meta.xml`

**Logic:**
- **After Insert:** If `LinkedEntityId` references a `Messenger_Message__c` or `Messenger_Attachment__c` record (check SObjectType), set `Has_File__c = true` and increment `File_Count__c` on the parent message.
- **After Delete:** Decrement `File_Count__c`. If count reaches 0, set `Has_File__c = false`.
- **Bulkified:** Collect all affected message IDs, query once, update once.

**Steps:**
1. Write test class first -- insert ContentVersion with FirstPublishLocationId, verify Has_File__c = true
2. Write test for delete scenario -- verify Has_File__c = false after last file removed
3. Write test for bulk scenario -- 200 ContentDocumentLinks in one transaction
4. Implement trigger + handler
5. Deploy and run: `sf apex run test --synchronous --class-names ContentDocumentLinkTriggerHandlerTest`
6. Commit: `feat: contentdocumentlink trigger for has_file denormalization`

---

### Task 2.3: Integration Test - Full Inbound Flow

**Agent:** `go-backend`
**Files:**
- Create: `internal/media/integration_test.go` (build tag: `integration`)

**Test scenario:**
1. Create a `Messenger_Message__c` record via REST API
2. Upload a test image (JPEG, 100 KB) as ContentVersion with FirstPublishLocationId
3. Verify ContentDocumentLink was auto-created
4. Verify `Has_File__c = true` on the message record
5. Verify `Content_Version_Id__c` is populated on the attachment record
6. Download the file via VersionData endpoint and verify binary matches

**Steps:**
1. Write integration test (requires scratch org connection)
2. Run: `go test ./internal/media/ -v -tags=integration -count=1`
3. Commit: `test: inbound contentversion integration test`

---

## Phase 3: LWC Display (Show Files in Chat)

**Agent:** `salesforce-dev`
**Dependencies:** Phase 2 (files exist in ContentVersion)
**Verification:** `sf apex run test --test-level RunLocalTests --wait 30` and manual scratch org verification

### Task 3.1: Update messengerChat to Display Images

**Agent:** `salesforce-dev`
**Files:**
- Modify: `lwc/messengerChat/messengerChat.html`
- Modify: `lwc/messengerChat/messengerChat.js`
- Modify: `lwc/messengerChat/messengerChat.css`

**Implementation:**
- Query `Messenger_Attachment__c` records where `Has_File__c = true` on the parent message
- Construct servlet URL: `/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB720BY480&versionId={Content_Version_Id__c}`
- Render `<img>` tags with lazy loading (`loading="lazy"`)
- Display only when `Has_File__c = true` (skip ContentDocumentLink query entirely)

**Steps:**
1. Add image rendering to message template (conditional on Has_File__c)
2. Implement lazy loading with intersection observer
3. Deploy and verify images render in scratch org
4. Commit: `feat: lwc image display via contentversion servlet urls`

---

### Task 3.2: Video, Voice, Document, and Sticker Display

**Agent:** `salesforce-dev`
**Files:**
- Modify: `lwc/messengerChat/messengerChat.html`
- Modify: `lwc/messengerChat/messengerChat.js`
- Create: `lwc/messengerMediaViewer/` (lightbox component)

**Implementation by type:**

| Type | Display | Interaction |
|---|---|---|
| Image | `<img>` with thumbnail rendition | Click: open in NavigationMixin lightbox |
| Video | Thumbnail + play button overlay | Click: download full file, then play in `<video>` |
| Voice | Custom audio player bar | Inline `<audio>` with waveform visualization |
| Document | File icon + filename + size | Click: download via NavigationMixin |
| Sticker | `<img>` (usually WebP/PNG) | No interaction needed |

**Steps:**
1. Implement type-based rendering in message template
2. Build messengerMediaViewer lightbox component using NavigationMixin
3. Implement download-first video UX with progress indicator
4. Deploy and verify each media type in scratch org
5. Commit: `feat: lwc multi-type media display with lightbox`

---

### Task 3.3: Album Grid Layout

**Agent:** `salesforce-dev`
**Files:**
- Modify: `lwc/messengerChat/messengerChat.html`
- Modify: `lwc/messengerChat/messengerChat.css`

**Implementation:**
- When `File_Count__c > 1` on a message, render attached images in a CSS grid
- Grid layout: 2-column for 2-4 images, 3-column for 5+ images
- Telegram media groups map to messages with multiple attachments

**Steps:**
1. Implement CSS grid layout for multi-file messages
2. Handle mixed media types (image + document in same message)
3. Deploy and verify album rendering
4. Commit: `feat: lwc album grid layout for multi-file messages`

---

## Phase 4: Outbound Flow (SF -> Telegram)

**Agent:** `go-backend` + `salesforce-dev`
**Dependencies:** Phase 1 (download adapter), MVP Phase 6.1 (outbound message sending)
**Verification:** `go test ./internal/media/... -v` and `sf apex run test`

### Task 4.1: File Upload in LWC

**Agent:** `salesforce-dev`
**Files:**
- Modify: `lwc/messengerChat/messengerChat.html`
- Modify: `lwc/messengerChat/messengerChat.js`

**Implementation:**
- Use `lightning-file-upload` component with `record-id` set to the current chat or message record
- Accept: images, videos, documents, voice memos
- On upload complete, fire a Platform Event or invoke Apex controller to notify Go middleware

**Steps:**
1. Add `lightning-file-upload` to chat compose area
2. Implement `onuploadfinished` handler that creates `Messenger_Attachment__c` record
3. Fire notification to Go middleware (via Apex callout to Go endpoint, or Platform Event)
4. Deploy and verify file upload in scratch org
5. Commit: `feat: lwc file upload with lightning-file-upload`

---

### Task 4.2: Go Download from ContentVersion + Send to Telegram

**Agent:** `go-backend`
**Files:**
- Create: `internal/media/outbound.go`
- Create: `internal/media/outbound_test.go`

**Flow:**
```
Platform Event / REST callback from SF
    -> Go receives notification with ContentVersionId + Telegram recipient
    -> Go downloads file via ContentVersion VersionData endpoint (streaming)
    -> Go sends file to Telegram via MTProto messages.sendMedia or Bot API sendDocument/sendPhoto
    -> Go updates Messenger_Message__c delivery status
```

**Steps:**
1. Write failing test with mock SF download + mock Telegram upload
2. Implement streaming download-then-upload pipeline (no full file buffering)
3. Handle large files (chunked Telegram upload for files > 10 MB via MTProto)
4. Run: `go test ./internal/media/ -v -run TestOutbound`
5. Commit: `feat: outbound media pipeline contentversion to telegram`

---

### Task 4.3: Platform Event or Callback for Upload Notification

**Agent:** `salesforce-dev` + `go-backend`
**Files:**
- Create or modify: Platform Event for media upload notification
- Create: Apex controller method for outbound media callout

**Implementation:**
- Option A: New Platform Event `Media_Upload__e` with ContentVersionId and recipient fields
- Option B: Apex `@future(callout=true)` or Queueable that calls Go REST endpoint
- Recommendation: Use Option B (Queueable callout) for reliability and retry capability

**Steps:**
1. Implement Apex Queueable that sends ContentVersionId to Go middleware
2. Go endpoint receives notification and triggers outbound media flow
3. Test with mock callout in Apex test class
4. Commit: `feat: outbound media upload notification flow`

---

## Phase 5: Cleanup and Polish

**Agent:** `go-backend` + `salesforce-dev`
**Dependencies:** Phases 1-4 complete
**Verification:** Full test suite pass, scratch org smoke test

### Task 5.1: Remove R2 Adapter Code from Go

**Agent:** `go-backend`
**Files to remove:**
- `internal/media/r2.go` (if created)
- `internal/media/r2_test.go` (if created)
- Any R2/S3 client dependencies in `go.mod`

**Steps:**
1. Remove R2 client code and tests
2. Remove AWS SDK S3 dependencies: `go mod tidy`
3. Verify: `go build ./...` and `go test ./... -count=1`
4. Commit: `chore: remove r2 adapter code`

---

### Task 5.2: Remove R2/Workers Infrastructure Config

**Agent:** `go-backend`
**Files to remove:**
- `workers/media-proxy/` directory (if created)
- Any `wrangler.toml` configuration
- R2-related environment variables from config

**Steps:**
1. Remove Worker source and config files
2. Remove R2 env vars from `internal/config/config.go`
3. Update config tests
4. Commit: `chore: remove cloudflare workers and r2 config`

---

### Task 5.3: Remove CSP Trusted Sites for R2

**Agent:** `salesforce-dev`
**Files:**
- Remove any CSP Trusted Site metadata for R2 endpoint domain

**Steps:**
1. Remove CSP Trusted Site XML metadata (if created)
2. Deploy to scratch org
3. Commit: `chore: remove r2 csp trusted sites`

---

### Task 5.4: Deprecate Media_URL__c Fields

**Agent:** `salesforce-dev`
**Files:**
- Modify: `Messenger_Message__c` object metadata
- Modify: `Messenger_Attachment__c` object metadata

**Approach:**
- Do NOT delete `Media_URL__c` fields. Managed packages cannot delete fields after release.
- Mark as deprecated in field description: "Deprecated by ADR-20. Use Content_Version_Id__c."
- Remove from page layouts and list views.
- Set field-level security to hidden for all profiles.
- Keep for upgrade safety -- existing customers may have data in these fields.

**Steps:**
1. Update field descriptions to indicate deprecation
2. Remove from layouts
3. Update FLS to hidden
4. Deploy and verify
5. Commit: `chore: deprecate media_url fields in favor of contentversion`

---

### Task 5.5: Update Permission Sets

**Agent:** `salesforce-dev`
**Files:**
- Modify: `Messenger_Admin.permissionset-meta.xml`
- Modify: `Messenger_User.permissionset-meta.xml`

**Changes:**
- Add Read access to ContentVersion and ContentDocumentLink for Messenger_User
- Add Create access to ContentVersion for Messenger_Admin (file upload)
- Add new fields (`Has_File__c`, `File_Count__c`, `Content_Version_Id__c`, etc.) to both permission sets

**Steps:**
1. Update permission set XML
2. Deploy and verify permissions
3. Commit: `feat: update permission sets for contentversion access`

---

### Task 5.6: Storage Monitoring Dashboard

**Agent:** `salesforce-dev`
**Files:**
- Create: `lwc/messengerStorageMonitor/`

**Implementation:**
- Apex controller: query `System.OrgLimits.getMap()` for `FileStorageMB` (used/max)
- LWC: display storage usage bar chart with configurable alert thresholds
- Admin-only component (check permission set in connectedCallback)
- Display recommendations when usage exceeds 70%, 85%, 95%

**Steps:**
1. Write Apex controller with test class
2. Build LWC with usage bar and alert indicators
3. Deploy and verify in scratch org
4. Commit: `feat: admin lwc storage monitoring dashboard`

---

## Doc Update Plan

The following files require updates to reflect ADR-20. Listed by priority. **Do NOT edit these files as part of this migration plan -- they are tracked here for separate execution.**

| File | What Changes | Priority |
|---|---|---|
| `CLAUDE.md` (root) | "Never store media in Salesforce. Always Cloudflare R2 + Worker CDN" -> "All media stored in Salesforce ContentVersion. See ADR-20." | P0 |
| `.claude/skills/media-pipeline.md` | Complete rewrite: R2 architecture -> ContentVersion architecture. Remove all R2 code samples, CORS/CSP sections, Worker sections. Add ContentVersion upload/download patterns, servlet URLs, LWC display patterns. | P0 |
| `.claude/skills/question-resolver.md` | Principle #6 "Media in R2" -> "Media in ContentVersion -- all media stored in Salesforce Files" | P0 |
| `MessageForge.Documentation/architecture/architecture.md` | Update data flow diagram: remove R2/Workers boxes. Update "What Lives Where" section: remove "Cloudflare R2: All media files" and "media_files" PG table. Add ContentVersion to Salesforce section. | P1 |
| `MessageForge.Documentation/architecture/adr.md` | Append ADR-20 row to table. Update ADR-3 and ADR-14 entries with "(superseded by ADR-20)" note. Add full ADR-20 section below ADR-19. | P1 |
| `MessageForge.Documentation/reference/sf-data-model.md` | Add new fields: `Has_File__c`, `File_Count__c`, `Content_Version_Id__c`, `Content_Document_Id__c`, `Thumbnail_Content_Version_Id__c`, `File_Size_Bytes__c`, `Content_Type__c`. | P1 |
| `MessageForge.Documentation/reference/security-checklist.md` | Add ContentVersion CRUD/FLS checks. Add: "ContentVersion upload via REST API requires Create permission." Add: "ContentDocumentLink sharing model verification." | P1 |
| `MessageForge.Documentation/reference/postgres-schema.md` | Remove `media_files` table definition. Add note: "Media references stored in Salesforce ContentVersion (ADR-20), not PostgreSQL." | P1 |
| `MessageForge.Documentation/plans/mvp-implementation-plan.md` | Phase 4 (Media Pipeline): replace all R2 tasks with ContentVersion tasks. Update tech stack line. Update architecture description. | P1 |
| `MessageForge.Documentation/CLAUDE.md` | Update architecture overview: remove R2/Workers from diagram. Update Key Decisions table: ADR-3 and ADR-14 superseded. | P1 |
| `CLAUDE.md` (root) Structure table | Remove "R2" from Backend stack column. | P2 |
| `CLAUDE.md` (root) Skills Index | Update media-pipeline skill description: "Upload/download/display media via R2" -> "Upload/download/display media via ContentVersion" | P2 |
| `.claude/skills/data-ingestion.md` | If references media_files table or R2 URLs in payload examples, update to ContentVersion references. | P2 |
| `.claude/skills/outbound-failure-sync.md` | If references R2 download in outbound flow, update to ContentVersion download. | P2 |
| `MessageForge.Documentation/reference/deployment-guide.md` | Remove Cloudflare R2 and Workers setup sections. | P2 |
| `MessageForge.Documentation/plans/appexchange-onboarding.md` | Update external component list: remove R2/Workers. Simplify security review section (fewer external dependencies to document). | P2 |
| `MessageForge.Documentation/plans/future-multi-messenger.md` | If references R2 for media handling in future platforms, update to ContentVersion. | P2 |
| `MessageForge.Documentation/reviews/2026-03-29-risk-register.md` | R15 (media orphan growth): mark as RESOLVED. Add new risks: API call budget for media (HIGH), customer storage quota (MEDIUM). | P2 |
| `MessageForge.Documentation/architecture/adr-20-draft.md` | Add note at top: "This draft was superseded by the accepted ADR-20. See `adr-20-media-storage-pivot.md`." | P3 |
| `MessageForge.Documentation/research/salesforce-files-FINAL-analysis.md` | Add note at top: "Decision overridden. ADR-20 was accepted on 2026-03-30. See `architecture/adr-20-media-storage-pivot.md`." | P3 |
