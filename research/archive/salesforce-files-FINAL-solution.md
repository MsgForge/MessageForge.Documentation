# Salesforce Files Media Storage — Complete Solution Document

**Date:** 2026-03-30
**Decision:** ACCEPTED (ADR-20). All media in Salesforce ContentVersion.
**Supersedes:** ADR-3 (R2 + Workers), ADR-14 (SigV4 presigned URLs)

---

## 1. Executive Summary

All media files (photos, videos, voice, documents, stickers) will be stored in Salesforce ContentDocument/ContentVersion instead of Cloudflare R2. This eliminates 10+ external infrastructure components (R2 bucket, Workers, CDN proxy, HMAC signing, SigV4 presigned URLs, CSP Trusted Sites, CORS, PostgreSQL `media_files` table) and replaces them with standard Salesforce Files APIs. Every major AppExchange messaging app (SMS-Magic, Mogli, Heymarket, Sinch Engage, 360 SMS, and Salesforce's own Digital Engagement) uses this exact pattern — SMS-Magic notably **migrated away** from external storage TO Salesforce Files. The Go middleware uploads media via REST API multipart (`POST /sobjects/ContentVersion` with `FirstPublishLocationId`), and LWC displays files via same-origin servlet URLs (`/sfc/servlet.shepherd/`) with zero CORS/CSP configuration needed.

---

## 2. Architecture Design

> Full details: `research/salesforce-files-architecture-design.md`

### 2A. Inbound Data Flow (Telegram → Salesforce)

```
Telegram Server
    │ MTProto getFile / Bot API file download
    ▼
Go Middleware
    │ 1. Download binary from Telegram
    │ 2. Generate thumbnail (ffmpeg for video)
    │ 3. POST multipart to Salesforce REST API
    │    (ContentVersion with FirstPublishLocationId = Message record)
    ▼
Salesforce REST API
    │ 4. ContentVersion created (VersionData = binary)
    │ 5. ContentDocument auto-created
    │ 6. ContentDocumentLink auto-created → Messenger_Message__c
    ▼
ContentDocumentLink Trigger
    │ 7. Sets Has_File__c = true, increments File_Count__c
    ▼
LWC (same-origin /sfc/servlet.shepherd/)
    │ 8. Renders thumbnail inline in chat bubble
    │ 9. Click opens NavigationMixin filePreview
```

### 2B. Outbound Data Flow (Salesforce → Telegram)

```
LWC Chat UI
    │ 1. Agent uses <lightning-file-upload> to attach file
    ▼
Salesforce (automatic)
    │ 2. ContentVersion + ContentDocumentLink created
    │ 3. Trigger fires → Has_File__c = true
    ▼
Apex Queueable (callout to Go)
    │ 4. Sends ContentVersionId + recipient to Go
    ▼
Go Middleware
    │ 5. GET .../ContentVersion/{id}/VersionData (streaming)
    │ 6. Streams to Telegram via MTProto/Bot API
    ▼
Platform Event → LWC (delivery status)
```

### 2C. System Diagram

```
                +------------------+
                |   Messaging      |
                |   Platforms      |
                |   (Telegram)     |
                +--------+---------+
                         │
                +--------▼---------+         +-----------------+
                |     Go           |◄───────►|   PostgreSQL    |
                |   Middleware     |         |   (sessions,    |
                +--+------+--------+         |    queues)      |
                   │      │                  +-----------------+
          +--------+      +--------+
          │                        │
  +-------▼------+         +------▼----------+
  |  Salesforce   |         |   Centrifugo    |
  |  REST API     |         |   (WebSocket)   |
  +-------+-------+         +------+----------+
          │                        │
  +-------▼-------+               │
  | ContentVersion |               │
  | (file binary)  |               │
  +-------+-------+               │
          │                        │
  +-------▼-----------------------▼+
  |           LWC (Browser)         |
  |  /sfc/servlet.shepherd/...      |
  |  (same-origin, no CORS needed)  |
  +---------------------------------+
```

**Key change:** Cloudflare R2, Workers, CDN proxy, presigned URLs, and CORS/CSP for external media domains are ALL eliminated.

### 2D. Data Model Changes

**New fields on `Messenger_Message__c`:**

| API Name | Type | Default | Purpose |
|---|---|---|---|
| `tgint__Has_File__c` | Checkbox | `false` | Fast filter for file queries. Set by CDL trigger. |
| `tgint__Contains_Inbound_Files__c` | Checkbox | `false` | Direction-aware flag. Set by Go. |
| `tgint__File_Count__c` | Number(3,0) | `0` | Denormalized count. Updated by CDL trigger. |

**New fields on `Messenger_Attachment__c`:**

| API Name | Type | Length | Purpose |
|---|---|---|---|
| `tgint__Content_Version_Id__c` | Text | 18 | Direct CV reference for URL construction |
| `tgint__Content_Document_Id__c` | Text | 18 | For NavigationMixin filePreview |
| `tgint__Thumbnail_CV_Id__c` | Text | 18 | Video/sticker thumbnail CV reference |
| `tgint__File_Size_Bytes__c` | Number(10,0) | — | File size for display + monitoring |
| `tgint__Content_Type__c` | Text | 100 | MIME type for rendering logic |

**Deprecated fields (cannot delete from 2GP):**

| Field | Action |
|---|---|
| `Media_URL__c` (Message + Attachment) | Mark deprecated in description, hide from layouts, stop writing |
| `Thumbnail_URL__c` (Attachment) | Same deprecation pattern |

### 2E. LWC Display by Media Type

| Type | Display | Interaction | URL Pattern |
|---|---|---|---|
| **Photo** | `<img>` thumbnail (720x480 rendition) | Click → NavigationMixin filePreview | `/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB720BY480&versionId=` |
| **Video** | Poster thumbnail + play overlay | <25MB: inline `<video>`, 25-100MB: click-to-download-then-play, >100MB: download button | `/sfc/servlet.shepherd/version/download/` |
| **Voice** | `<audio>` player + waveform SVG | Inline playback | `/sfc/servlet.shepherd/version/download/` |
| **Document** | SLDS doctype icon + name + size | Click → NavigationMixin filePreview, download button | filePreview + `/version/download/` |
| **Sticker** | `<img>` at 200x200px | None (decorative) | `/sfc/servlet.shepherd/version/download/` |
| **Album** | CSS grid (2-col for 2-4, 3-col for 5+) | Each cell clickable → filePreview with navigation | Same as photo |

### 2F. API Call Budget Per Media Message

| Operation | API Calls | Notes |
|---|---|---|
| Upload ContentVersion (file) | 1 | Multipart POST |
| Query ContentDocumentId | 1 | Can batch for albums |
| Upload thumbnail CV (video only) | 0-1 | Only for video/sticker |
| Create Messenger_Attachment__c | 1 | JSON POST |
| **Total (photo)** | **3** | |
| **Total (video)** | **4** | Includes thumbnail upload |
| **Total (album of 10)** | **12** | 10 uploads + 1 batch query + 1 batch attachment create |

### 2G. Components Added/Removed

| Component | Change | Location |
|---|---|---|
| R2 bucket | **REMOVE** | Cloudflare |
| Cloudflare Workers | **REMOVE** | Cloudflare |
| HMAC URL signing | **REMOVE** | Go middleware |
| SigV4 presigned URL gen | **REMOVE** | Go middleware |
| CSP Trusted Sites for R2 | **REMOVE** | SF Setup |
| CORS config for R2 | **REMOVE** | Cloudflare + SF |
| `media_files` PG table | **REMOVE** | PostgreSQL |
| Media orphan cleanup job | **REMOVE** | Go middleware |
| `R2Client` Go code | **REMOVE** | `internal/media/r2.go` |
| Worker source | **REMOVE** | `workers/media-proxy/` |
| ContentVersion upload adapter | **ADD** | Go `internal/salesforce/contentversion.go` |
| ContentVersion download adapter | **ADD** | Go (streaming) |
| Media upload queue + rate limiter | **ADD** | Go `internal/media/queue.go` |
| ContentDocumentLink trigger | **ADD** | Apex trigger + handler |
| Storage monitoring LWC | **ADD** | Admin dashboard |
| `Has_File__c`, `File_Count__c` | **ADD** | Messenger_Message__c |
| `Content_Version_Id__c` + related | **ADD** | Messenger_Attachment__c |

---

## 3. Verified Technical Facts

> Full details: `research/salesforce-files-deep-research.md`

| Claim | Verified? | Value | Source | Confidence |
|---|---|---|---|---|
| Max file size (multipart upload) | **YES** | 2 GB | [SF REST API Docs](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_sobject_insert_update_blob.htm) | HIGH |
| Max file size (Base64 JSON body) | **YES** | ~37.5 MB | Same source | HIGH |
| Each file = 1 API call (no batching) | **YES** | Composite/Bulk don't support blobs | Same source | HIGH |
| FirstPublishLocationId with custom objects | **YES** | Polymorphic, accepts any record ID | [ContentVersion Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_contentversion.htm) | HIGH |
| Auto-creates ContentDocumentLink | **YES** | Via FirstPublishLocationId | Same source | HIGH |
| Enterprise storage (100 users) | **YES** | 210 GB (10 GB base + 2 GB/user) | [SF Files Storage](https://help.salesforce.com/s/articleView?id=experience.files_storage.htm) | HIGH |
| Additional storage cost | **YES** | $5/GB/month | [CloudFiles](https://www.cloudfiles.io/blog/salesforce-file-storage-cost), [XfilesPro](https://www.xfilespro.com/detailed-analysis-of-salesforce-file-storage-cost/) | HIGH |
| Storage grace period | **YES** | Up to ~110% before block | [XfilesPro](https://www.xfilespro.com/salesforce-file-storage-limit-exceeded-some-use-cases-tips-to-prevent-hitting-storage-limits/) | MED |
| Enterprise API limit (50 users) | **YES** | 150,000/day | [SF API Limits Blog](https://developer.salesforce.com/blogs/2024/11/api-limits-and-monitoring-your-api-usage) | HIGH |
| Unlimited API limit (50 users) | **YES** | 350,000/day | Same source | HIGH |
| Additional API calls cost | **YES** | $25/month per 10K/day | Same source | HIGH |
| Video playback with seeking | **YES** | Works (SalesforceLabs VideoViewer) | [GitHub/SalesforceLabs/VideoViewer](https://github.com/SalesforceLabs/VideoViewer) | MED |
| All competitors use ContentVersion | **YES** | SMS-Magic, Mogli, Heymarket, Sinch, 360 SMS | Multiple AppExchange listings | HIGH |
| SMS-Magic migrated FROM external | **YES** | Moved TO Salesforce Files | [SMS-Magic Docs](https://www.sms-magic.com/docs/salesforce/knowledge-base/send-a-multimedia-message/) | HIGH |
| SF Digital Engagement uses CV | **YES** | Spring '24 file attachments | [SF Release Notes](https://help.salesforce.com/s/articleView?id=release-notes.rn_miaw_file_attachments.htm) | HIGH |
| CRUD/FLS = #1 review failure | **YES** | Mandatory for AppExchange | [Noltic Guide](https://noltic.com/stories/how-to-pass-salesforce-appexchange-security-review) | HIGH |
| CDL query requires filter | **YES** | Must filter on LinkedEntityId or ContentDocumentId | [CDL Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_contentdocumentlink.htm) | HIGH |
| `addError()` doesn't work in CDL trigger | **YES** | Known limitation | [SF Help](https://help.salesforce.com/s/articleView?id=000381623) | HIGH |
| Security review timeline | **YES** | 4-8 weeks | [Noltic](https://noltic.com/stories/how-to-pass-salesforce-appexchange-security-review) | MED |
| Auto-generated thumbnails | **YES** | Images + PDFs only (not video) | Training data + community | MED |

---

## 4. Risk Solutions

> Full details: `research/salesforce-files-risk-solutions.md`

**Every risk has a concrete solution. No risk is a blocker.**

| # | Risk | Severity | Solution | Confidence |
|---|---|---|---|---|
| R1 | **Storage growth — org fills up** | HIGH | Retention batch job (configurable days), compression pipeline (Go-side), deduplication (SHA-256), storage monitoring LWC, admin alerts at 70/85/95% | HIGH |
| R2 | **API call budget exhaustion** | CRITICAL | PostgreSQL upload queue, per-org token bucket rate limiter, real-time `Sforce-Limit-Info` header monitoring, deferred upload mode, ISV Connected App 5x increase | HIGH |
| R3 | **Large video upload timeout/memory** | HIGH | Streaming via `io.Pipe` + temp file (never buffer in RAM), configurable size cap (default 100 MB), retry with exponential backoff, dynamic timeout per file size | HIGH |
| R4 | **CDL trigger conflicts** | MEDIUM | Async Queueable Apex (200 SOQL headroom), defensive governor checks, chunked bulk insert (5 per Queueable), pre-install diagnostic | HIGH |
| R5 | **AppExchange security review** | MEDIUM | `WITH SECURITY_ENFORCED` on all SOQL, CRUD checks before all DML, file type allowlist (block executables), `ShareType='I'` on all CDLs, Code Analyzer scan | HIGH |
| R6 | **Chat UI performance (50+ images)** | MEDIUM | 20-message pagination, denormalized `Content_Version_Id__c` (0 extra SOQL for thumbnails), `loading="lazy"`, virtual scrolling for 500+ threads | HIGH |
| R7 | **Video playback (no range requests)** | HIGH | Three-tier UX: <25MB auto-play inline, 25-100MB thumbnail+click-to-play, >100MB download button. Go-side ffmpeg thumbnail extraction for all videos. | MED |
| R8 | **Field deprecation (can't delete from 2GP)** | LOW | Mark deprecated in description, hide from layouts/FLS, stop writing, keep for upgrade safety | HIGH |
| R9 | **Multi-org storage variance** | HIGH | Runtime `System.OrgLimits.getMap()` check, Go-side pre-upload storage gate via `/limits` endpoint, edition-aware admin dashboard, graceful degradation to metadata-only | HIGH |
| R10 | **GDPR file deletion** | MEDIUM | Cascade deletion utility (`GDPRDeletionService`), configurable auto-delete trigger on message deletion, orphan cleanup batch, `Database.emptyRecycleBin()` for permanent purge | HIGH |

### Key Math Models

**Storage (R1) — Enterprise 100 users, 210 GB pool:**
- Base scenario (200 media/day, 500KB avg): 3 GB/month → **70 months** to exhaust
- Realistic mix (200/day, 1.41 MB weighted avg): 8.46 GB/month → **25 months** to exhaust
- Aggressive (500/day, 1 MB avg): 15 GB/month → **14 months** to exhaust
- All scenarios manageable with retention policies + additional storage purchase

**API Budget (R2) — Enterprise 100 users, 100K calls/day:**
- Light (50 media/day): 55 calls → 0.07%
- Heavy (5,000/day): 5,500 calls → 6.9%
- Very heavy (20,000/day): 22,000 calls → 27.5%
- ISV Connected App 5x multiplier available → 500K calls/day

---

## 5. Implementation Roadmap

> Full details: `plans/media-storage-migration-plan.md`

### Phase 1: Foundation (Data Model + Go Adapter)
| Task | Agent | Key Deliverable |
|---|---|---|
| 1.1 Add new SF fields | `salesforce-dev` | `Has_File__c`, `Content_Version_Id__c`, etc. |
| 1.2 Go CV upload adapter | `go-backend` | Multipart upload with `FirstPublishLocationId` |
| 1.3 Go CV download adapter | `go-backend` | Streaming download via `VersionData` |
| 1.4 Media upload queue | `go-backend` | Rate-limited queue with API budget monitoring |

### Phase 2: Inbound Flow (Telegram → SF Files)
| Task | Agent | Key Deliverable |
|---|---|---|
| 2.1 Wire inbound pipeline | `go-backend` | Download-from-Telegram → upload-to-ContentVersion |
| 2.2 CDL trigger | `salesforce-dev` | `Has_File__c` + `File_Count__c` denormalization |
| 2.3 Integration test | `go-backend` | Full round-trip with scratch org |

### Phase 3: LWC Display (Show Files in Chat)
| Task | Agent | Key Deliverable |
|---|---|---|
| 3.1 Image display | `salesforce-dev` | Thumbnails via servlet URLs, lazy loading |
| 3.2 Multi-type media | `salesforce-dev` | Video, voice, document, sticker + lightbox |
| 3.3 Album grid | `salesforce-dev` | CSS grid for multi-file messages |

### Phase 4: Outbound Flow (SF → Telegram)
| Task | Agent | Key Deliverable |
|---|---|---|
| 4.1 File upload in LWC | `salesforce-dev` | `lightning-file-upload` in compose area |
| 4.2 Go download + send | `go-backend` | Streaming CV download → Telegram upload |
| 4.3 Upload notification | `salesforce-dev` + `go-backend` | Queueable callout to Go |

### Phase 5: Cleanup & Polish
| Task | Agent | Key Deliverable |
|---|---|---|
| 5.1 Remove R2 adapter | `go-backend` | Delete R2 client code + AWS SDK deps |
| 5.2 Remove Workers config | `go-backend` | Delete wrangler.toml, R2 env vars |
| 5.3 Remove CSP Trusted Sites | `salesforce-dev` | Delete R2 CSP metadata |
| 5.4 Deprecate Media_URL__c | `salesforce-dev` | Mark deprecated, hide from layouts |
| 5.5 Update permission sets | `salesforce-dev` | Add CV/CDL access to permission sets |
| 5.6 Storage monitoring | `salesforce-dev` | Admin LWC with `System.OrgLimits` |

**Estimated effort:** ~38 days sequential, ~22 days with Go/Apex parallelization.

---

## 6. ADR-20 Reference

**File:** `architecture/adr-20-media-storage-pivot.md`

- **Status:** Accepted
- **Date:** 2026-03-30
- **Supersedes:** ADR-3 (R2 + Workers), ADR-14 (SigV4 presigned URLs)
- **Decision:** ALL media stored in Salesforce ContentDocument/ContentVersion. R2 removed entirely.
- **Trade-offs accepted:** Higher client storage cost (feature not bug), download-first video UX, API call consumption (monitoring + throttling)
- **5 risks with mitigations** included in the ADR itself

---

## 7. Documentation Update Checklist

> From `plans/media-storage-migration-plan.md` (Doc Update Plan section)

| Priority | File | What Changes |
|---|---|---|
| **P0** | `CLAUDE.md` (root) | "Never store media in Salesforce" → "All media in ContentVersion. See ADR-20." |
| **P0** | `.claude/skills/media-pipeline.md` | Complete rewrite: R2 → ContentVersion architecture |
| **P0** | `.claude/skills/question-resolver.md` | Principle #6 "Media in R2" → remove |
| **P1** | `architecture/architecture.md` | Remove R2/Workers from diagram, update data flows |
| **P1** | `architecture/adr.md` | Append ADR-20 row, mark ADR-3/14 superseded |
| **P1** | `reference/sf-data-model.md` | Add new fields (Has_File, CV IDs, etc.) |
| **P1** | `reference/security-checklist.md` | Add ContentVersion CRUD/FLS checks |
| **P1** | `reference/postgres-schema.md` | Remove `media_files` table |
| **P1** | `plans/mvp-implementation-plan.md` | Replace R2 tasks with ContentVersion tasks |
| **P1** | `MessageForge.Documentation/CLAUDE.md` | Update architecture diagram, Key Decisions table |
| **P2** | `CLAUDE.md` Structure table | Remove "R2" from Backend stack |
| **P2** | `.claude/skills/data-ingestion.md` | Update payload examples |
| **P2** | `.claude/skills/outbound-failure-sync.md` | Update outbound flow references |
| **P2** | `plans/appexchange-onboarding.md` | Remove R2/Workers from external components |
| **P2** | `reviews/2026-03-29-risk-register.md` | Mark R15 resolved, add new media risks |
| **P3** | `architecture/adr-20-draft.md` | Add superseded note |
| **P3** | `research/salesforce-files-FINAL-analysis.md` | Add decision-overridden note |

---

## 8. Verified Limits Table

| Limit | Value | Source | Workaround if Hit |
|---|---|---|---|
| Max file (REST multipart) | 2 GB | [SF REST API Docs](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_sobject_insert_update_blob.htm) | Configurable size cap (default 100 MB) |
| Max file (Base64 JSON body) | ~37.5 MB | Same | Always use multipart |
| API calls/day (Enterprise 100u) | 100,000 | [SF Limits](https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_api.htm) | ISV 5x multiplier, API add-on packs ($25/10K/day) |
| API calls/day (Unlimited 100u) | 600,000 | Same | Effectively uncapped for most use cases |
| File storage (Enterprise, 100u) | 210 GB | [SF Files Storage](https://help.salesforce.com/s/articleView?id=experience.files_storage.htm) | Additional storage $5/GB/month |
| Storage grace period | ~110% | [XfilesPro](https://www.xfilespro.com/salesforce-file-storage-limit-exceeded-some-use-cases-tips-to-prevent-hitting-storage-limits/) | Retention policy + monitoring |
| Apex sync heap | 6 MB | [SF Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm) | All binary handling in Go, never in Apex |
| Apex async heap | 12 MB | Same | Same approach |
| SOQL queries (sync) | 100 | Same | Denormalization eliminates file queries |
| SOQL queries (async) | 200 | Same | CDL trigger uses async when possible |
| DML statements | 150 | Same | Chunked bulk insert (5 per Queueable) |
| CDL query filter requirement | Must filter on LinkedEntityId or ContentDocumentId | [CDL Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_contentdocumentlink.htm) | Always query by message record ID |
| CDL max links per document | ~2,000 (unverified) | Community sources | Irrelevant — we create 1-2 links per file |
| REST API timeout | ~10 min (empirical) | Community observed | Dynamic timeout per file size, retry with backoff |
| `addError()` in CDL trigger | Does NOT work | [SF Help](https://help.salesforce.com/s/articleView?id=000381623) | Log errors, don't block file creation |

---

## 9. Unresolved Items (Scratch Org Testing Needed)

1. **HTTP Range request support** — Does `/sfc/servlet.shepherd/version/download/` return `Accept-Ranges: bytes`? Test with `curl -I`. If yes, video seeking works natively. If no, the three-tier video UX (R7 solution) handles it.

2. **FirstPublishLocationId with `tgint__` namespaced records** — Confirmed polymorphic in general, but test specifically with `tgint__Messenger_Message__c` record IDs via REST API.

3. **CDL trigger firing on REST API uploads** — Verify that uploading ContentVersion with `FirstPublishLocationId` via Go REST API fires `after insert` trigger on the auto-created ContentDocumentLink.

4. **File Upload and Download Security defaults** — Check if `.mp4`, `.webm`, `.mov` are set to "Execute in Browser" or "Download" by default in a fresh scratch org.

5. **Storage consumption accuracy** — Upload 100 test files of various sizes and verify SF's reported storage matches sum of file sizes (confirm thumbnails/renditions don't count).

6. **Concurrent upload throughput** — Benchmark 10, 50, 100 parallel ContentVersion uploads for rate limit discovery.

7. **ContentVersion insert response format** — Verify exact response JSON and whether `ContentDocumentId` is returned directly or requires a follow-up query.

8. **NavigationMixin filePreview for video** — Test whether the native preview modal handles video playback or only shows a download button.

---

## 10. Next Action

**The implementation is ready to begin.**

- **ADR-20** is written and accepted: `architecture/adr-20-media-storage-pivot.md`
- **Migration plan** is ready: `plans/media-storage-migration-plan.md`
- **Start with Phase 1** of the migration plan (Foundation: data model + Go adapter)

### Recommended first steps:
1. Update P0 documentation (`CLAUDE.md`, `media-pipeline.md`, `question-resolver.md`)
2. Append ADR-20 summary row to `architecture/adr.md`
3. Begin Phase 1, Task 1.1: Add new fields to Salesforce objects
4. Begin Phase 1, Task 1.2: Build Go ContentVersion upload adapter (can run in parallel with 1.1)

### Source Documents
| Document | Purpose |
|---|---|
| `research/salesforce-files-deep-research.md` | Web-verified API details, competitors, limits |
| `research/salesforce-files-architecture-design.md` | Complete architecture with data flows, data model, LWC design |
| `research/salesforce-files-risk-solutions.md` | 10 risks with concrete solutions and code samples |
| `architecture/adr-20-media-storage-pivot.md` | Accepted ADR (supersedes ADR-3, ADR-14) |
| `plans/media-storage-migration-plan.md` | 5-phase, 16-task implementation roadmap |
