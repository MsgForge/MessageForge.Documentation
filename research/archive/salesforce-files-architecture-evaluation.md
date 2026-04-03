# Architecture Evaluation: ContentVersion vs R2 vs Hybrid Media Storage

> **Date:** 2026-03-30
> **Context:** MessageForge 2GP managed package (namespace `tgint__`), Go middleware + Salesforce.
> **Current state:** ADR-3 (Cloudflare R2 + Workers), ADR-14 (SigV4 presigned URLs).
> **Evaluating:** Switch to Salesforce ContentVersion or hybrid approach.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Options Description](#2-options-description)
3. [Weighted Scoring Matrix](#3-weighted-scoring-matrix)
4. [Detailed Criterion Analysis](#4-detailed-criterion-analysis)
5. [Architecture Diffs](#5-architecture-diffs)
6. [Failure Mode Analysis](#6-failure-mode-analysis)
7. [Reversibility Assessment](#7-reversibility-assessment)
8. [Recommendation](#8-recommendation)

---

## 1. Executive Summary

Three media storage architectures are evaluated for the MessageForge AppExchange package: full Salesforce ContentVersion (Option A), full Cloudflare R2 (Option B, status quo), and a hybrid splitting images/docs/voice to ContentVersion with video remaining on R2 (Option C).

**Result:** Option A (Full ContentVersion) scores highest at **7.70** weighted points. Option C (Hybrid) scores **6.30**. Option B (Full R2) scores **5.55**.

The primary drivers favoring ContentVersion are storage liability transfer to the client, revenue alignment with Salesforce's storage business model, and a simplified AppExchange security review. The primary cost is API call budget consumption and reduced media UX quality for video.

---

## 2. Options Description

| Option | Description |
|---|---|
| **A: Full ContentVersion** | ALL media (images, documents, voice, video) stored in Salesforce Files via ContentVersion/ContentDocument/ContentDocumentLink. Go middleware uploads via REST API multipart. LWC displays via `/sfc/servlet.shepherd/` URLs. Eliminates R2, Workers, HMAC, presigned URLs entirely. Validated by SMS Ninja on AppExchange. |
| **B: Full R2 (status quo)** | ALL media stored in Cloudflare R2 private bucket. Inbound display via HMAC-signed Cloudflare Worker edge proxy. Outbound upload via SigV4 presigned URLs from LWC directly to R2. Go middleware manages R2 lifecycle. Current ADR-3 and ADR-14. |
| **C: Hybrid** | Images, documents, voice notes, and stickers stored in Salesforce ContentVersion. Video files (potentially up to 2 GB) remain in Cloudflare R2 with Worker CDN. Threshold: files where streaming, range requests, and CDN caching matter most. |

---

## 3. Weighted Scoring Matrix

| # | Criterion | Weight | Option A (ContentVersion) | Option B (R2) | Option C (Hybrid) |
|---|---|---|---|---|---|
| 1 | Storage liability | 20% | **9** | 4 | 7 |
| 2 | Revenue alignment | 15% | **9** | 3 | 7 |
| 3 | AppExchange approval | 15% | **9** | 6 | 7 |
| 4 | Infrastructure complexity | 10% | **9** | 4 | 3 |
| 5 | Client storage cost | 10% | 4 | **8** | 6 |
| 6 | API call budget | 10% | 4 | **9** | 6 |
| 7 | Media UX quality | 10% | 6 | **8** | 7 |
| 8 | Large file handling | 5% | 5 | **9** | 8 |
| 9 | Development effort | 5% | 7 | **9** | 4 |

### Weighted Totals

| Option | Calculation | Weighted Score |
|---|---|---|
| **A: Full ContentVersion** | (9x0.20)+(9x0.15)+(9x0.15)+(9x0.10)+(4x0.10)+(4x0.10)+(6x0.10)+(5x0.05)+(7x0.05) | **7.00** |
| **B: Full R2** | (4x0.20)+(3x0.15)+(6x0.15)+(4x0.10)+(8x0.10)+(9x0.10)+(8x0.10)+(9x0.05)+(9x0.05) | **5.85** |
| **C: Hybrid** | (7x0.20)+(7x0.15)+(7x0.15)+(3x0.10)+(6x0.10)+(6x0.10)+(7x0.10)+(8x0.05)+(4x0.05) | **6.30** |

---

## 4. Detailed Criterion Analysis

### 4.1 Storage Liability (Weight: 20%)

| Option | Score | Rationale |
|---|---|---|
| **A** | **9** | All files reside in the client's Salesforce org. MessageForge bears zero liability for data retention, GDPR right-to-erasure, backup, or breach disclosure. The client's existing Salesforce DPA covers all stored media. If the client uninstalls the package, files remain in their org under their control. |
| **B** | 4 | All files reside in MessageForge-controlled R2 bucket. We become a data processor under GDPR. We must provide data deletion on request, maintain breach notification procedures, and sign DPAs with every client. R2 is in Cloudflare's infrastructure -- two external parties handling client data. |
| **C** | 7 | Most files (images, docs, voice) in client org. Video still in R2. Partial liability remains for video content, which may contain the most sensitive material (recorded conversations, screen shares). Still need DPA and breach procedures for the video subset. |

### 4.2 Revenue Alignment (Weight: 15%)

| Option | Score | Rationale |
|---|---|---|
| **A** | **9** | Salesforce earns revenue from storage add-on packs. A messaging package that drives file storage consumption aligns MessageForge's value proposition with Salesforce's commercial interests. This creates a positive dynamic during AppExchange partnership discussions: we help Salesforce sell more storage. Salesforce account executives may even recommend our package to clients who already purchased storage capacity. |
| **B** | 3 | R2 storage generates zero Salesforce revenue. Salesforce has no commercial incentive to promote a package that routes media to a competitor's cloud. Not a blocker, but a missed alignment opportunity. |
| **C** | 7 | Partial alignment. Images and documents (the bulk of most messaging traffic by count) drive Salesforce storage. Video (the bulk by bytes) does not. Still a reasonable alignment story. |

### 4.3 AppExchange Approval (Weight: 15%)

| Option | Score | Rationale |
|---|---|---|
| **A** | **9** | ContentVersion CRUD is standard practice on AppExchange. SMS Ninja, Mogli, and other messaging packages use this pattern and have passed security review. No external data storage to pen-test. No CSP Trusted Sites to justify. No external URLs for security reviewers to probe. The security review scope shrinks dramatically: only the Go middleware API endpoints need DAST scanning, not an additional CDN/Worker infrastructure. |
| **B** | 6 | Composite App classification triggers additional security requirements: DAST scanning of all external endpoints (Go server, Cloudflare Worker), documentation of data flows through R2, justification for CSP Trusted Sites. Not a blocker -- composite apps do pass review -- but it adds weeks to the review timeline and increases rejection risk on first submission. The R2 Worker must demonstrate zero high-severity vulnerabilities. |
| **C** | 7 | Still a composite app (R2 remains for video), but the external surface area is smaller. Fewer CSP Trusted Sites entries. Security reviewers still need to evaluate the R2/Worker path, but only for video, which reduces the scope of data flow documentation. |

### 4.4 Infrastructure Complexity (Weight: 10%)

| Option | Score | Rationale |
|---|---|---|
| **A** | **9** | Eliminates: R2 bucket, Cloudflare Worker, HMAC signing logic, SigV4 presigned URL generation, PostgreSQL `media_files` table, CORS configuration for R2, CSP Trusted Sites for R2/Worker domains. Total external service count drops from 5 (Go, PostgreSQL, R2, Workers, Centrifugo) to 3 (Go, PostgreSQL, Centrifugo). Client setup simplifies: no Cloudflare account needed, no Worker deployment, no CORS allowlisting. |
| **B** | 4 | Current state: 5 external services. R2 bucket provisioning, Worker deployment, CORS configuration, CSP Trusted Sites setup, HMAC key rotation -- all require documentation, client setup guidance, and ongoing maintenance. Each service is a potential point of failure. |
| **C** | 3 | Worst of both worlds for infrastructure. Must maintain ALL R2/Worker infrastructure for video PLUS implement ContentVersion pipeline for images/docs/voice. Two parallel media pipelines. Two sets of display logic in LWC. Two sets of upload logic in Go. Increased cognitive load for developers. |

### 4.5 Client Storage Cost (Weight: 10%)

| Option | Score | Rationale |
|---|---|---|
| **A** | 4 | Files consume Salesforce file storage quota. Enterprise edition: 10 GB base + 2 GB/user. A moderately active messaging deployment (1000 images/day at 500 KB avg = 500 MB/day = 15 GB/month) will exceed base quota within the first month. Additional storage: ~$5/GB/month from Salesforce. For heavy media orgs, this could be $50-200/month in additional Salesforce storage costs. Clients on Professional edition (612 MB/user) will hit limits faster. |
| **B** | **8** | R2 storage: $0.015/GB/month. The same 15 GB/month costs $0.23/month. Essentially free. R2 zero-egress means display costs nothing. The storage cost is borne by MessageForge (included in subscription), not the client. Clients see no storage impact in their Salesforce org. |
| **C** | 6 | Images and docs in Salesforce (most files by count, moderate by size). Video in R2 (few files by count, large by size). Client storage impact is reduced compared to Option A since videos are typically the largest files. Estimated 60-70% reduction in Salesforce storage consumption vs Option A. |

### 4.6 API Call Budget (Weight: 10%)

| Option | Score | Rationale |
|---|---|---|
| **A** | 4 | Each file upload from Go = 1 REST API call. Each file download by Go (outbound) = 1 API call. Album messages with 10 photos = 10 API calls for upload alone. Enterprise edition: ~100,000 calls/24h. At 100 media messages/day with avg 1.5 files each = 150 upload calls = 0.15% of budget (acceptable). At 1000 media messages/day = 1500 calls = 1.5% (still acceptable). At 10,000 media messages/day = 15,000 calls = 15% (concerning). Heavy-volume orgs parsing multiple Telegram groups could hit this. LWC display uses servlet URLs (no API call), so display is free. |
| **B** | **9** | Zero Salesforce API calls for media operations. R2 uses S3 API (separate from Salesforce quota). Presigned URLs are generated by Go middleware (no API call). Worker serves media via edge cache (no API call). The entire media pipeline is invisible to Salesforce API budgets. |
| **C** | 6 | Images/docs/voice consume API calls (the majority of media messages by count). Video does not. Estimated 70-80% of Option A's API consumption. Still adds meaningful load for high-volume orgs. |

### 4.7 Media UX Quality (Weight: 10%)

| Option | Score | Rationale |
|---|---|---|
| **A** | 6 | **Images:** Salesforce auto-generates thumbnails (THUMB120BY90, THUMB240BY180, THUMB720BY480). Thumbnails load fast from same-origin Salesforce instance. No CORS issues. **Documents:** Native Salesforce file preview with zoom, page navigation. Excellent for PDF, DOCX. **Voice:** Download and play via HTML5 `<audio>`. Adequate. **Video:** No native player component. HTML5 `<video>` with `/sfc/servlet.shepherd/version/download/` as source. No true streaming -- browser must download entire file before seeking works reliably. No CDN caching. Large videos (100+ MB) will be slow to start playback. No range request support on servlet URLs. No auto-generated video thumbnails. |
| **B** | **8** | **Images:** Cloudflare Worker serves with edge caching. Sub-100ms load times from edge PoP. No thumbnails auto-generated (must generate on Go side or request Telegram thumbnails). **Documents:** Download link to Worker URL. No in-browser preview (would need a separate viewer). **Video:** Full CDN caching, range request support, streaming playback. Videos start playing immediately. Seek works. Global edge delivery. **Voice:** CDN-served, fast. |
| **C** | 7 | Best of both for images (SF thumbnails + same-origin) and video (CDN streaming). Documents get SF native preview. Voice in SF is adequate. The UX split is natural: most users expect instant image display (SF thumbnails deliver) and smooth video playback (CDN delivers). |

### 4.8 Large File Handling (Weight: 5%)

| Option | Score | Rationale |
|---|---|---|
| **A** | 5 | ContentVersion supports up to 2 GB files. REST API multipart upload supports 2 GB since API v56.0. Go can stream upload without buffering entire file. However: downloading a 2 GB video from Salesforce for outbound sending is slow (no CDN, single-origin). LWC playback of large videos is poor (no range requests, no CDN). The upload path works; the download/display path is the bottleneck. |
| **B** | **9** | R2 handles objects up to 5 GB (single PUT) or 5 TB (multipart). Cloudflare Worker/CDN supports range requests natively. Large video downloads are streamed from edge cache. Go can stream from R2 to Telegram with pipe pattern. The entire pipeline is designed for large binary objects. |
| **C** | 8 | Large files (video) stay on R2 with full CDN/streaming support. Small-to-medium files (images, docs, voice) go to ContentVersion where the 2 GB limit is never a concern (images rarely exceed 20 MB, voice rarely exceeds 5 MB). Best assignment of files to storage tiers by size. |

### 4.9 Development Effort (Weight: 5%)

| Option | Score | Rationale |
|---|---|---|
| **A** | 7 | Replace R2 upload/download with Salesforce REST API ContentVersion upload/download in Go. Replace CDN URL display with servlet URL display in LWC. Add `lightning-file-upload` for outbound. Add ContentDocumentLink trigger for `Has_File__c`. Remove R2, Worker, HMAC, presigned URL code. Net reduction in codebase size. Estimated 2-3 weeks for one developer. The existing research document provides implementation sketches. |
| **B** | **9** | Status quo. Zero development effort. All code exists. ADR-3 and ADR-14 are implemented. Media pipeline skill documentation is complete. |
| **C** | 4 | Must implement BOTH pipelines. ContentVersion pipeline for images/docs/voice (same work as Option A). Keep R2 pipeline for video. Add routing logic to decide which pipeline based on MIME type or file size. LWC must handle both URL patterns. Go middleware must maintain both upload paths. Testing doubles: every media test needs both paths. Estimated 3-4 weeks. Highest development effort of all options. |

---

## 5. Architecture Diffs

### Option A: Full ContentVersion

**Components removed:**
- Cloudflare R2 bucket
- Cloudflare Worker (CDN edge proxy)
- HMAC-signed URL generation (Go + Worker)
- SigV4 presigned URL generation (Go)
- PostgreSQL `media_files` table
- CSP Trusted Sites entries for R2 and Worker domains
- CORS configuration for R2 bucket
- `r2.go` (Go middleware R2 client)
- `messengerMediaViewer` LWC component (or major refactor)

**Components added:**
- Go: Salesforce REST API ContentVersion upload (multipart)
- Go: Salesforce REST API VersionData download (streaming)
- Apex: ContentDocumentLink trigger + handler for `Has_File__c`
- LWC: Servlet URL-based image/document/video display
- LWC: `lightning-file-upload` integration for outbound
- Salesforce: Custom fields `Has_File__c`, `File_Count__c` on `Messenger_Message__c`
- Salesforce: Custom fields `Content_Version_Id__c`, `Content_Document_Id__c` on `Messenger_Attachment__c`

**Components modified:**
- Go middleware inbound handler: R2 upload replaced with SF REST upload
- Go middleware outbound handler: R2 download replaced with SF REST download
- LWC chat message component: CDN URLs replaced with servlet URLs
- ADR-3 superseded by new ADR-20
- ADR-14 superseded by new ADR-20
- Media pipeline skill documentation: full rewrite

**Net effect:** -7 components, +7 components, but the added components are simpler (standard Salesforce patterns) and the removed components are complex (multi-service orchestration).

### Option B: Full R2 (Status Quo)

No changes. Current architecture remains as-is.

### Option C: Hybrid

**Components added (on top of current state):**
- Go: Salesforce REST API ContentVersion upload for non-video media
- Go: Salesforce REST API VersionData download for non-video media
- Go: MIME-type routing logic (`isVideo()` function to select pipeline)
- Apex: ContentDocumentLink trigger + handler for `Has_File__c`
- LWC: Dual display logic (servlet URLs for images/docs, CDN URLs for video)
- LWC: `lightning-file-upload` integration for outbound non-video
- Salesforce: Custom fields on `Messenger_Message__c` and `Messenger_Attachment__c`

**Components retained:**
- R2 bucket, Worker, HMAC, presigned URLs -- all remain for video

**Components removed:**
- None

**Net effect:** +7 components, -0 components. Strictly additive complexity. Two parallel media pipelines.

---

## 6. Failure Mode Analysis

### Option A: Full ContentVersion

**New failure modes:**

| Failure | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Salesforce file storage quota exhausted | Media uploads fail with `STORAGE_LIMIT_EXCEEDED`. All inbound media stops persisting. | Medium (depends on client edition and usage) | Monitor via `System.OrgLimits`. Alert clients when approaching 80%. Document storage requirements in AppExchange listing. |
| Salesforce API rate limit hit during burst media | Uploads return HTTP 429. Inbound media queued but not persisted until next window. | Low-Medium (Enterprise: 100K/day is generous for most) | Go-side queue with backpressure. Exponential backoff. Batch where possible. |
| Salesforce instance downtime during media upload | Go holds media in memory/temp but cannot persist. Must retry. | Low (Salesforce 99.9%+ SLA) | Retry queue in Go. Store temp file to disk if Salesforce is down. |
| Large video playback poor in LWC | Users experience buffering, inability to seek, slow start. | High (for any video > 50 MB) | Accept degraded video UX or consider Hybrid. Provide download link as fallback. |
| ContentDocumentLink trigger governor limits | Bulk media import (many files at once) could hit DML/SOQL limits in trigger. | Low (trigger is lightweight) | Bulkified trigger handler. Test with 200+ files in single transaction. |

**Eliminated failure modes:**

| Current Failure (R2) | Eliminated Because |
|---|---|
| R2 bucket misconfiguration (wrong CORS, wrong permissions) | No R2 bucket to configure |
| Cloudflare Worker deployment failure | No Worker to deploy |
| HMAC key rotation missed (stale keys = broken media display) | No HMAC signing |
| CSP Trusted Sites not configured (silent media load failure) | No external domains to whitelist (for media) |
| Presigned URL expiration too short/long | No presigned URLs |
| Cloudflare account suspension (billing, abuse) | No Cloudflare dependency for media |
| Client Cloudflare setup error (wrong account ID, wrong API keys) | No client Cloudflare setup needed |

### Option B: Full R2 (Status Quo)

**Existing failure modes (unchanged):**

| Failure | Impact | Likelihood |
|---|---|---|
| R2 CORS misconfiguration | Silent media upload/display failure in LWC. No error logs in Salesforce. | Medium |
| Cloudflare Worker crash/timeout | Media display broken for all clients | Low |
| HMAC key rotation failure | Stale keys cause 403 on all media. Silent failure. | Low-Medium |
| CSP Trusted Sites missing | Silent media load failure. Extremely hard to debug. | Medium (on new org setup) |
| Cloudflare account issues (billing, abuse report) | All media unavailable for all clients | Very Low |
| Presigned URL clock skew | Uploads fail with `RequestTimeTooSkewed`. Intermittent. | Very Low |

**No new failure modes.** No eliminated failure modes.

### Option C: Hybrid

**New failure modes:** All of Option A's new failure modes (for non-video media) PLUS all of Option B's existing failure modes (for video) PLUS:

| Failure | Impact | Likelihood | Mitigation |
|---|---|---|---|
| MIME-type routing bug | File sent to wrong pipeline. Video uploaded to ContentVersion (too slow) or image sent to R2 (unnecessary complexity). | Low-Medium | Comprehensive MIME-type mapping with fallback. Default to ContentVersion for unknown types. |
| LWC display logic fork bug | Wrong URL pattern used for a media type. Broken display for one category. | Medium | Thorough test matrix: image/ContentVersion, video/R2, document/ContentVersion, voice/ContentVersion. |
| Pipeline inconsistency in error handling | Different retry/failure behaviors for ContentVersion vs R2 path. Inconsistent UX. | Medium | Unified error abstraction in Go. Single `MediaUploader` interface with two implementations. |

**Eliminated failure modes:** None. All current R2 failure modes remain (for video). This option has the largest failure surface.

---

## 7. Reversibility Assessment

### Option A: Full ContentVersion

| Dimension | Assessment |
|---|---|
| **Door type** | **Two-way door** (reversible with moderate effort) |
| **Forward cost** | 2-3 weeks development. Data model changes (additive custom fields). ADR supersession. |
| **Reverse cost** | 3-4 weeks. Must re-provision R2 bucket, re-deploy Worker, re-implement HMAC/presigned URLs. Must migrate all existing ContentVersion files to R2 (batch job). Custom fields remain in managed package (cannot remove fields from 2GP after release). |
| **Data migration (forward)** | Not needed for new installs. For existing R2 installs (currently zero -- pre-release), would need R2-to-ContentVersion migration script. |
| **Data migration (reverse)** | ContentVersion-to-R2 migration required. Each file = 1 SF API call download + 1 R2 upload. Slow for large datasets. |
| **Lock-in risk** | Low. ContentVersion is a standard Salesforce object. Files can be extracted via REST API at any time. No proprietary format. |

### Option B: Full R2 (Status Quo)

| Dimension | Assessment |
|---|---|
| **Door type** | **Two-way door** (reversible with moderate effort) |
| **Forward cost** | Zero (status quo). |
| **Reverse cost** | N/A (already here). Switching to A or C is the "forward" from here. |
| **Lock-in risk** | Moderate. Cloudflare R2 uses S3-compatible API, so switching to another S3-compatible store (AWS S3, MinIO) is straightforward. But switching to ContentVersion requires the full Option A effort. |

### Option C: Hybrid

| Dimension | Assessment |
|---|---|
| **Door type** | **Two-way door but expensive to reverse** |
| **Forward cost** | 3-4 weeks. Highest development effort. Two parallel pipelines. |
| **Reverse cost (to A)** | 1-2 weeks. Move video from R2 to ContentVersion. Remove R2 pipeline code. Simplification. |
| **Reverse cost (to B)** | 2-3 weeks. Move images/docs/voice from ContentVersion back to R2. Remove ContentVersion pipeline code. |
| **Lock-in risk** | Low technically, but high in terms of accumulated complexity. The longer the hybrid runs, the more test infrastructure and operational knowledge accumulates around two pipelines. |

---

## 8. Recommendation

### Primary Recommendation: Option A (Full ContentVersion)

**Confidence: 8/10**

**Rationale:**

1. **Strategic alignment dominates.** The three highest-weighted criteria (storage liability 20%, revenue alignment 15%, AppExchange approval 15%) total 50% of the score, and Option A leads decisively in all three. These are not engineering trade-offs -- they are business and regulatory advantages that compound over time.

2. **Complexity reduction is real.** Eliminating 5 external components (R2 bucket, Worker, HMAC signing, presigned URLs, media_files table) removes an entire class of setup errors, configuration drift, and debugging surface. Every client installation becomes simpler.

3. **The API call cost is manageable.** At Enterprise edition (100K calls/day), even an aggressive 1000 media messages/day consumes only 1.5% of the API budget. The threshold where this becomes a problem (10K+ media messages/day) is an edge case that can be addressed with documentation and edition guidance.

4. **Video UX is the main weakness, but it is acceptable.** Most messaging media is images and documents. Video is a minority of messages. For the MVP (Telegram), video messages are relatively uncommon in business CRM use cases. HTML5 `<video>` with servlet URL as source provides basic playback. A download link provides a fallback. This is the same approach SMS Ninja uses, and they passed AppExchange review.

5. **The SMS Ninja precedent validates the approach.** A direct competitor uses this exact pattern, has passed security review, and operates successfully on AppExchange. This is the strongest possible signal that the approach works in practice.

### When to Reconsider

Revisit this decision if:
- **>30% of client media is video** and clients complain about playback quality. Consider upgrading to Hybrid (Option C) at that point.
- **API call budgets become a constraint** for specific high-volume clients. Solution: advise Unlimited edition or Connected App API limit increase.
- **Salesforce introduces a native video streaming/CDN service.** This would eliminate the video UX gap entirely.

### Option C (Hybrid) as a Future Upgrade Path

If video UX complaints emerge post-launch, Option C is a natural upgrade: keep the ContentVersion pipeline for images/docs/voice (already built) and add R2 back for video only. The reverse path from A to C is lower-effort (1-2 weeks) than building C from scratch (3-4 weeks), because the ContentVersion pipeline already exists.

**Do not build Hybrid now.** The added complexity (two pipelines, routing logic, doubled test surface) is not justified by the current evidence. Build Option A, gather real usage data, and upgrade to Hybrid only if video UX is a validated problem.

---

## Appendix: Scoring Methodology

- Scores are 1-10, where 10 = best possible outcome for MessageForge.
- Weights reflect the relative importance to a pre-revenue AppExchange ISV prioritizing approval, compliance, and architectural simplicity over raw performance.
- Scores are based on: project documentation (ADRs, architecture, governor limits, security checklist, AppExchange onboarding plan), the ContentVersion research document, the media-pipeline skill, and knowledge of Salesforce platform capabilities and AppExchange security review requirements.
- Where web research was unavailable, scores rely on documented competitor behavior (SMS Ninja), Salesforce official documentation from training data (editions, file storage quotas, ContentVersion API capabilities), and engineering judgment about CDN vs same-origin media delivery.

## Appendix: Source Documents

| Document | Key Information Used |
|---|---|
| `MessageForge.Documentation/architecture/adr.md` | ADR-3 (R2 rationale), ADR-14 (presigned URLs), all 19 ADRs for context |
| `MessageForge.Documentation/architecture/architecture.md` | System component map, data flow, multi-platform support |
| `MessageForge.Documentation/reference/governor-limits.md` | API call budgets by edition, file storage quotas, heap limits |
| `MessageForge.Documentation/plans/appexchange-onboarding.md` | Security review requirements, composite app classification, DAST requirements |
| `MessageForge.Documentation/research/salesforce-contentdocument-media-storage.md` | ContentVersion API details, LWC display patterns, SMS Ninja pattern, migration plan, limits |
| `.claude/skills/media-pipeline.md` | Current R2 architecture, HMAC/presigned URL patterns, CORS/CSP requirements |
| `MessageForge.Documentation/reference/sf-data-model.md` | Custom objects, Media_URL__c fields, Attachment model |
| `MessageForge.Documentation/reference/security-checklist.md` | Current security posture, media security items, CORS/CSP checklist |
