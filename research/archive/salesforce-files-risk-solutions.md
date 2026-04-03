# Risk Register with Solutions -- Salesforce Files (ContentVersion) as Sole Media Storage

**Date:** 2026-03-30
**Status:** DECISION FINAL -- All media stored in Salesforce ContentDocument/ContentVersion
**Scope:** 10 risks with concrete solutions for MessageForge 2GP managed package (namespace `tgint__`)

---

## Executive Summary

This document accepts Salesforce ContentVersion as the sole media storage backend and provides a concrete solution for every identified risk. No risk is a blocker when the correct mitigation is implemented. The key architectural investments are:

1. **Go-side queue with rate limiter** -- prevents API call exhaustion by controlling upload pace
2. **Storage monitoring dashboard** -- gives admins visibility and triggers auto-cleanup
3. **Streaming upload pipeline** -- handles large files without memory pressure
4. **Video UX adaptation** -- download-first pattern for large videos, auto-play for small ones
5. **Defensive Apex patterns** -- governor-safe triggers, FLS/CRUD enforcement from day one

### Risk Overview Table

| ID | Risk | Severity | Solution Confidence | Key Solution |
|---|---|---|---|---|
| R1 | Storage Growth -- Client Org Fills Up | HIGH | HIGH | Retention policies + storage monitoring + compression |
| R2 | API Call Budget -- Heavy Media Usage | CRITICAL | HIGH | Go-side queue + rate limiter + API budget monitor |
| R3 | Large Video Upload -- Timeout/Memory | HIGH | HIGH | Streaming via io.Pipe + temp file + configurable size cap |
| R4 | ContentDocumentLink Trigger Conflicts | MEDIUM | HIGH | Async Queueable + defensive governor checks |
| R5 | AppExchange Security Review -- File Upload | MEDIUM | HIGH | FLS/CRUD checklist + file type allowlist |
| R6 | Chat UI Performance -- Many Images | MEDIUM | HIGH | Lazy loading + pagination + denormalized IDs |
| R7 | Video Playback -- Large File Load Times | MEDIUM | HIGH | Size-tiered playback + SalesforceLabs VideoViewer pattern |
| R8 | Managed Package Field Deprecation | LOW | HIGH | Deprecation-in-place + hide from layouts |
| R9 | Multi-Org Storage Variance | HIGH | HIGH | Runtime OrgLimits check + graceful degradation |
| R10 | GDPR File Deletion | MEDIUM | HIGH | Cascade deletion utility + configurable retention |

---

## R1: Storage Growth -- Client Org Fills Up

### Scenario

A subscriber org with 100 Enterprise users receives 200 media messages/day (aggregate). Average file size: 500 KB (weighted mix of photos, documents, occasional videos). Media accumulates indefinitely because ContentVersion records persist unless explicitly deleted.

### Math Model

**Enterprise edition storage pool:**
- Base: 10 GB
- Per user: 2 GB x 100 users = 200 GB
- Total pool: **210 GB**

**Base scenario (200 media/day, 500 KB avg):**
```
200 files/day x 0.5 MB = 100 MB/day = 3 GB/month
210 GB / 3 GB/month = 70 months (~5.8 years)
```

**Aggressive scenario (500 media/day, 1 MB avg incl. videos):**
```
500 files/day x 1 MB = 500 MB/day = 15 GB/month
210 GB / 15 GB/month = 14 months (~1.2 years)
```

**Realistic mix with videos (200 media/day, 1.41 MB weighted avg):**
```
Photos 70% @ 0.3 MB = 0.21 MB weighted
Docs   20% @ 1.0 MB = 0.20 MB weighted
Videos 10% @ 10 MB  = 1.00 MB weighted
Weighted avg: 1.41 MB

200 x 1.41 MB = 282 MB/day = 8.46 GB/month
210 GB / 8.46 GB/month = 25 months (~2 years)
```

### Impact

- New file uploads fail with `STORAGE_LIMIT_EXCEEDED`
- Existing files remain accessible (no data loss)
- Other DML operations continue (storage limit only blocks new file/data creation)
- Unlike API limits, storage does NOT auto-reset -- requires admin action

### SOLUTION: Storage Monitoring + Auto-Cleanup + Compression

**1. Managed Package Storage Dashboard (LWC)**

Ship a Lightning component that shows:
- Total org file storage capacity (via `System.OrgLimits.getMap().get('FileStorageMB')`)
- Storage consumed by MessageForge files specifically (SOQL aggregate on ContentVersion WHERE CreatedBy matches package user)
- Projected exhaustion date based on 30-day growth trend
- One-click "clean up old media" button

**2. Configurable Retention Policy**

Custom Metadata Type: `tgint__Media_Retention__mdt`
- `Retention_Days__c`: Number of days to keep media (default: 180)
- `Auto_Delete_Enabled__c`: Boolean toggle (default: false, admin must opt in)
- `Delete_Threshold_Percent__c`: Only auto-delete when storage > N% full (default: 80)

Batch Apex scheduled job (runs daily):
```apex
global class MediaRetentionBatch implements Database.Batchable<SObject>, Schedulable {
    global Database.QueryLocator start(Database.BatchableContext bc) {
        Integer retentionDays = getRetentionDays(); // from Custom Metadata
        Date cutoff = Date.today().addDays(-retentionDays);
        return Database.getQueryLocator([
            SELECT Id FROM ContentDocument
            WHERE CreatedDate < :cutoff
            AND Id IN (SELECT ContentDocumentId FROM ContentDocumentLink
                       WHERE LinkedEntityId IN (SELECT Id FROM tgint__Messenger_Message__c))
        ]);
    }
    global void execute(Database.BatchableContext bc, List<ContentDocument> scope) {
        if (Schema.sObjectType.ContentDocument.isDeletable()) {
            delete scope;
            Database.emptyRecycleBin(scope);
        }
    }
}
```

**3. Go-Side Compression Before Upload**

- JPEG: Re-encode at quality 80, max dimension 1920px (reduces avg photo from 300 KB to ~150 KB)
- PNG: Convert to JPEG if no transparency needed
- Video: Optionally transcode to H.264 720p (configurable, off by default)
- Documents: Pass through unchanged

**4. Deduplication**

Before uploading, compute SHA-256 hash of file content. Query existing ContentVersions by hash (stored in `tgint__File_Hash__c` custom field on `Messenger_Attachment__c`). If duplicate found, reuse existing ContentDocumentLink.

**5. Admin Alert Pipeline**

- Salesforce sends email at 75%, 90%, 95% storage utilization (built-in)
- Managed package adds: Platform Event `tgint__Storage_Alert__e` published when Go middleware detects high usage via `/services/data/v62.0/limits` endpoint
- Go middleware checks storage on every 100th upload and logs warning at 70%

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| Storage dashboard LWC | `force-app/main/default/lwc/storageHealth/` | 2 days |
| Retention batch job | `force-app/main/default/classes/MediaRetentionBatch.cls` | 1 day |
| Custom Metadata Type | `force-app/main/default/customMetadata/Media_Retention__mdt` | 0.5 days |
| Go compression middleware | `internal/media/compress.go` | 1 day |
| Deduplication check | `internal/salesforce/dedup.go` | 0.5 days |
| **Total** | | **5 days** |

---

## R2: API Call Budget -- Heavy Media Usage

### Scenario

Each file upload to ContentVersion consumes 1 REST API call. Composite API and Bulk API 2.0 do NOT support binary blob uploads. In high-volume media scenarios, file uploads alone can exhaust the org's 24-hour API limit, blocking ALL middleware operations.

### Math Model

**Enterprise API budget (100 users):**
```
Base: 100,000 calls/24h (or 100 users x 1,000 = 100,000)
Non-media overhead (messages, config, status): ~20,000/day
Available for file uploads: ~80,000/day
```

**Usage tiers:**

| Tier | Media/day | API calls (file only) | % of 80K budget | Verdict |
|---|---|---|---|---|
| Light | 50 | 55 | 0.07% | Safe |
| Medium | 500 | 550 | 0.7% | Safe |
| Heavy | 5,000 | 5,500 | 6.9% | Safe |
| Very Heavy | 20,000 | 22,000 | 27.5% | Caution |
| Extreme | 50,000 | 55,000 | 68.8% | Danger |

**Peak hour (80% in 8 business hours, Heavy tier):**
```
5,000/day x 0.8 = 4,000 in 8 hours = 500/hour = 8.3/minute
```

### Impact

- HTTP 403 `REQUEST_LIMIT_EXCEEDED` on ALL REST API calls
- Blocks not just uploads but message queries, status updates, config reads
- Resets on rolling 24-hour basis (not midnight reset)

### SOLUTION: Go-Side Upload Queue + Rate Limiter + Budget Monitor

**1. PostgreSQL Upload Queue**

All file uploads go through a persistent queue in PostgreSQL, never directly to Salesforce:

```sql
CREATE TABLE media_upload_queue (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id      TEXT NOT NULL,
    file_path   TEXT NOT NULL,        -- temp file on disk
    file_name   TEXT NOT NULL,
    mime_type   TEXT NOT NULL,
    file_size   BIGINT NOT NULL,
    message_id  TEXT NOT NULL,         -- SF Messenger_Message__c ID
    file_hash   TEXT NOT NULL,         -- SHA-256 for dedup
    status      TEXT DEFAULT 'pending', -- pending, uploading, complete, failed, deferred
    attempts    INT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT now(),
    completed_at TIMESTAMPTZ
);
CREATE INDEX idx_queue_status ON media_upload_queue(org_id, status, created_at);
```

**2. Token Bucket Rate Limiter (per org)**

```go
type OrgRateLimiter struct {
    mu        sync.Mutex
    orgID     string
    budget    int64   // remaining API calls for this 24h window
    maxBudget int64   // org's total API limit
    uploads   *rate.Limiter // controls upload pace
}

// Configure based on org edition:
// Enterprise: 80,000 available -> 55 uploads/minute max (80K / 24h / 60min)
// With 50% safety margin: 27 uploads/minute
func NewOrgRateLimiter(orgID string, dailyBudget int64) *OrgRateLimiter {
    safeRate := float64(dailyBudget) * 0.5 / (24 * 60) // 50% of budget, spread over 24h
    return &OrgRateLimiter{
        orgID:     orgID,
        budget:    dailyBudget,
        maxBudget: dailyBudget,
        uploads:   rate.NewLimiter(rate.Limit(safeRate), 10), // burst of 10
    }
}
```

**3. Real-Time API Budget Monitoring**

Parse the `Sforce-Limit-Info` response header on every Salesforce API call:
```
Sforce-Limit-Info: api-usage=5000/100000
```

When usage exceeds 70%: reduce upload rate by 50%
When usage exceeds 85%: pause uploads entirely, queue in PostgreSQL
When usage exceeds 95%: pause ALL non-critical API calls

**4. Deferred Upload Mode**

When API budget is constrained:
- Queue files in PostgreSQL with `status = 'deferred'`
- Store file on local disk (temp storage)
- Upload during off-peak hours (configurable per org timezone)
- Send Platform Event to notify Salesforce UI: "Media pending upload -- will appear shortly"

**5. Connected App API Increase**

As an ISV partner, request increased API limits through Salesforce support:
- Standard Connected App: 1x org limit
- ISV partner Connected App: up to 5x org limit
- For Enterprise 100 users: 100K x 5 = 500K calls/day

**6. API Call Add-On Packages**

Document for customers: Salesforce sells API call add-on packs:
- 25,000 additional API calls/day: ~$25/month
- Unlimited Edition: effectively uncapped API calls

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| PostgreSQL queue table + migration | `migrations/003_media_queue.sql` | 0.5 days |
| Queue worker goroutine | `internal/media/queue_worker.go` | 2 days |
| Per-org rate limiter | `internal/ratelimit/org_limiter.go` | 1 day |
| API budget monitor (header parser) | `internal/salesforce/budget.go` | 0.5 days |
| Deferred upload scheduler | `internal/media/deferred.go` | 1 day |
| **Total** | | **5 days** |

---

## R3: Large Video Upload -- Timeout/Memory

### Scenario

A Telegram premium user sends a 500 MB video. Go middleware downloads it from Telegram, then must upload to Salesforce REST API as multipart form-data. Risk: HTTP timeout (Salesforce REST API empirically allows ~10 minutes for large uploads), memory pressure if buffered in RAM, and no resume capability on failure.

### Math Model

**Upload time at various network speeds (Go server to Salesforce):**

| File Size | 10 Mbps | 50 Mbps | 100 Mbps |
|---|---|---|---|
| 10 MB | 8 sec | 1.6 sec | 0.8 sec |
| 50 MB | 40 sec | 8 sec | 4 sec |
| 100 MB | 80 sec | 16 sec | 8 sec |
| 500 MB | 400 sec (6.7 min) | 80 sec | 40 sec |
| 2 GB | 1600 sec (27 min) | 320 sec (5.3 min) | 160 sec (2.7 min) |

**Salesforce limits:**
- Max file per ContentVersion: **2 GB** (REST API v56.0+)
- REST API server-side timeout: **10 minutes** (600,000ms; documented as `REQUEST_RUNNING_TOO_LONG` status code) -- [Source](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/resources_limits.htm)
- Go HTTP client default timeout: configurable

**Memory: buffering 500 MB in Go = 500 MB heap allocation. Unacceptable.**

### Impact

- Upload timeout: file lost, must retry from scratch
- Memory pressure: Go process OOM-killed by OS
- No resume: partial uploads are discarded by Salesforce

### SOLUTION: Streaming Pipeline + Temp File + Size Cap + Retry

**1. Two-Phase Download-Then-Upload via Temp File**

Never buffer large files in memory. Stream from Telegram to temp file, then stream from temp file to Salesforce:

```go
func handleLargeMedia(ctx context.Context, tgFile io.Reader, fileSize int64,
    sfUploader *SalesforceUploader) error {

    // Phase 1: Stream from Telegram to temp file
    tmpFile, err := os.CreateTemp("", "mf-media-*.tmp")
    if err != nil {
        return fmt.Errorf("create temp: %w", err)
    }
    defer os.Remove(tmpFile.Name())
    defer tmpFile.Close()

    written, err := io.Copy(tmpFile, tgFile)
    if err != nil {
        return fmt.Errorf("download to temp: %w", err)
    }
    tmpFile.Seek(0, io.SeekStart)

    // Phase 2: Stream from temp file to Salesforce multipart
    pr, pw := io.Pipe()
    writer := multipart.NewWriter(pw)

    go func() {
        defer pw.Close()
        // Write JSON metadata part
        metaPart, _ := writer.CreatePart(jsonHeader())
        json.NewEncoder(metaPart).Encode(metadata)
        // Stream binary part from temp file
        filePart, _ := writer.CreatePart(binaryHeader(fileName))
        io.Copy(filePart, tmpFile)
        writer.Close()
    }()

    // Upload with extended timeout
    client := &http.Client{Timeout: 15 * time.Minute}
    req, _ := http.NewRequestWithContext(ctx, "POST", sfURL, pr)
    req.Header.Set("Content-Type", writer.FormDataContentType())
    // Note: Content-Length unknown with pipe; Salesforce accepts chunked transfer
    resp, err := client.Do(req)
    // ... handle response
}
```

**2. Configurable Size Cap**

Custom Metadata Type: `tgint__Media_Settings__mdt`
- `Max_Upload_Size_MB__c`: Default 100 MB (admin can increase to 2048)
- Files exceeding cap: store metadata-only record in SF with `tgint__Oversized__c = true`
- Chat UI shows: "Video (487 MB) -- too large for storage. Available in Telegram."

**3. Retry with Exponential Backoff**

```go
func uploadWithRetry(ctx context.Context, upload func() error) error {
    backoff := []time.Duration{5*time.Second, 15*time.Second, 60*time.Second}
    for attempt, delay := range backoff {
        err := upload()
        if err == nil {
            return nil
        }
        if isTimeoutError(err) || isRetryableHTTPError(err) {
            log.Warnf("upload attempt %d failed: %v, retrying in %s", attempt+1, err, delay)
            time.Sleep(delay)
            continue
        }
        return err // non-retryable error
    }
    return fmt.Errorf("upload failed after %d attempts", len(backoff))
}
```

**4. HTTP Client Timeout Configuration**

```go
// Per-file timeout based on file size
func uploadTimeout(fileSize int64) time.Duration {
    // Base: 30 seconds + 1 minute per 100 MB
    base := 30 * time.Second
    perHundredMB := time.Duration(fileSize/(100*1024*1024)) * time.Minute
    timeout := base + perHundredMB
    if timeout > 15*time.Minute {
        timeout = 15 * time.Minute // hard cap
    }
    return timeout
}
```

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| Streaming upload with temp file | `internal/media/uploader.go` | 2 days |
| Size cap + oversized handling | `internal/media/size_check.go` | 0.5 days |
| Retry logic | `internal/media/retry.go` | 0.5 days |
| Dynamic timeout | `internal/salesforce/client.go` | 0.5 days |
| **Total** | | **3.5 days** |

---

## R4: ContentDocumentLink Trigger -- Conflicts with Other Packages

### Scenario

A subscriber org has other installed packages (document management, compliance tools) or customer-written triggers on `ContentDocumentLink`. When MessageForge bulk-inserts media files (e.g., album with 10 photos), all triggers fire in the same transaction, sharing the 100 SOQL / 150 DML governor limit pool.

### Impact

- Governor limit exhaustion in shared transaction
- DML exceptions that roll back MessageForge's file creation
- Unexpected side effects (notifications, workflows, process builder flows)

### SOLUTION: Async Processing + Defensive Governor Checks

**1. Queueable Apex for File Creation**

Never create ContentVersion records in synchronous context. Always use Queueable Apex:

```apex
public class FileCreationQueueable implements Queueable {
    private List<ContentVersion> filesToCreate;

    public FileCreationQueueable(List<ContentVersion> files) {
        this.filesToCreate = files;
    }

    public void execute(QueueableContext ctx) {
        // Async context: 200 SOQL, 60s CPU -- double headroom
        if (Schema.sObjectType.ContentVersion.isCreateable()) {
            insert filesToCreate;
        }
    }
}
```

Benefits:
- Async governor limits: 200 SOQL (vs 100 sync), 60s CPU (vs 10s)
- Isolates from synchronous trigger chain
- Other package triggers still fire but with more headroom

**2. Defensive Governor Check Before DML**

```apex
public class GovernorSafe {
    public static Boolean canInsertSafely(Integer recordCount) {
        // Reserve 20% of limits for other packages
        Integer soqlRemaining = Limits.getLimitQueries() - Limits.getQueries();
        Integer dmlRemaining = Limits.getLimitDmlStatements() - Limits.getDmlStatements();

        // Estimate: each ContentVersion insert may trigger 5 SOQL from other packages
        Integer estimatedSoqlCost = recordCount * 5;
        Integer estimatedDmlCost = recordCount * 2;

        return (soqlRemaining > estimatedSoqlCost * 1.2)
            && (dmlRemaining > estimatedDmlCost * 1.2);
    }
}
```

**3. Chunked Bulk Insert**

For album messages (10+ photos), split into chunks of 5:

```apex
public static void insertFilesInChunks(List<ContentVersion> allFiles) {
    Integer chunkSize = 5;
    for (Integer i = 0; i < allFiles.size(); i += chunkSize) {
        Integer endIdx = Math.min(i + chunkSize, allFiles.size());
        List<ContentVersion> chunk = new List<ContentVersion>();
        for (Integer j = i; j < endIdx; j++) {
            chunk.add(allFiles[j]);
        }
        System.enqueueJob(new FileCreationQueueable(chunk));
    }
}
```

**4. Pre-Install Diagnostic**

Provide an install-time diagnostic Apex script that checks for existing ContentDocumentLink triggers:

```apex
// Run in subscriber org before install
List<ApexTrigger> cdlTriggers = [
    SELECT Name, Body FROM ApexTrigger
    WHERE TableEnumOrId = 'ContentDocumentLink' AND Status = 'Active'
];
System.debug('Found ' + cdlTriggers.size() + ' active ContentDocumentLink triggers');
```

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| FileCreationQueueable | `force-app/main/default/classes/FileCreationQueueable.cls` | 1 day |
| GovernorSafe utility | `force-app/main/default/classes/GovernorSafe.cls` | 0.5 days |
| Chunked insert logic | `force-app/main/default/classes/MediaFileService.cls` | 0.5 days |
| Pre-install diagnostic | `scripts/check-cdl-triggers.apex` | 0.5 days |
| **Total** | | **2.5 days** |

---

## R5: AppExchange Security Review -- File Upload Requirements

### Scenario

AppExchange security review rejects the package for: missing FLS checks on ContentVersion queries, missing CRUD verification before insert/delete, allowing dangerous file types, or cross-channel file visibility leaks.

### Impact

- Rejection delays AppExchange listing by 2-4 weeks per resubmission
- Known reason for rejection: FLS enforcement is the #1 cause of security review failures

### SOLUTION: Security Compliance Checklist + Code Patterns

**1. FLS Enforcement on Every SOQL**

All ContentVersion/ContentDocumentLink queries MUST use `WITH SECURITY_ENFORCED`:

```apex
// CORRECT -- passes security review
List<ContentDocumentLink> links = [
    SELECT ContentDocumentId, ContentDocument.Title, ContentDocument.FileType
    FROM ContentDocumentLink
    WHERE LinkedEntityId = :messageId
    WITH SECURITY_ENFORCED
];

// ALSO CORRECT -- stripInaccessible for DML results
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.READABLE,
    [SELECT Id, Title, FileType FROM ContentVersion WHERE Id IN :versionIds]
);
List<ContentVersion> safeVersions = decision.getRecords();
```

**2. CRUD Checks Before Every DML**

```apex
public class SecureFileOps {
    public static void createFile(ContentVersion cv) {
        if (!Schema.sObjectType.ContentVersion.isCreateable()) {
            throw new SecurityException('Insufficient permissions to create files');
        }
        insert cv;
    }

    public static void deleteFile(ContentDocument cd) {
        if (!Schema.sObjectType.ContentDocument.isDeletable()) {
            throw new SecurityException('Insufficient permissions to delete files');
        }
        delete cd;
    }

    public static void linkFile(ContentDocumentLink cdl) {
        if (!Schema.sObjectType.ContentDocumentLink.isCreateable()) {
            throw new SecurityException('Insufficient permissions to link files');
        }
        insert cdl;
    }
}
```

**3. File Type Allowlist**

```apex
public class FileTypeValidator {
    private static final Set<String> ALLOWED_EXTENSIONS = new Set<String>{
        // Images
        'jpg', 'jpeg', 'png', 'gif', 'webp', 'bmp', 'svg',
        // Documents
        'pdf', 'doc', 'docx', 'xls', 'xlsx', 'ppt', 'pptx', 'txt', 'csv',
        // Audio
        'mp3', 'ogg', 'wav', 'aac', 'm4a',
        // Video
        'mp4', 'webm', 'mov', 'avi', 'mkv'
    };

    private static final Set<String> BLOCKED_EXTENSIONS = new Set<String>{
        'exe', 'bat', 'sh', 'cmd', 'ps1', 'vbs', 'jar', 'msi', 'dll', 'com'
    };

    public static Boolean isAllowed(String fileName) {
        String ext = fileName.substringAfterLast('.').toLowerCase();
        return ALLOWED_EXTENSIONS.contains(ext) && !BLOCKED_EXTENSIONS.contains(ext);
    }
}
```

**4. Cross-Channel Visibility Control**

```apex
// All ContentDocumentLinks use Inferred sharing
ContentDocumentLink cdl = new ContentDocumentLink();
cdl.ContentDocumentId = contentDocId;
cdl.LinkedEntityId = messageRecordId;
cdl.ShareType = 'I';            // Inferred from record sharing rules
cdl.Visibility = 'AllUsers';    // Visible to all who can see the record
```

With OWD Private on `Messenger_Chat__c`, files are only visible to users with access to the chat record.

**5. Security Review Checklist**

Before submission, verify:
- [ ] All SOQL on ContentVersion/ContentDocumentLink uses `WITH SECURITY_ENFORCED`
- [ ] All DML preceded by `isCreateable()` / `isDeletable()` checks
- [ ] File type validation blocks executable extensions
- [ ] `ShareType = 'I'` on all ContentDocumentLinks
- [ ] No SOQL injection (no string concatenation in queries)
- [ ] No `VersionData` blob queried in list contexts
- [ ] Unit tests run as limited-permission user to verify FLS enforcement
- [ ] Salesforce Code Analyzer (PMD/Graph Engine) passes with zero FLS violations

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| SecureFileOps utility class | `force-app/main/default/classes/SecureFileOps.cls` | 1 day |
| FileTypeValidator | `force-app/main/default/classes/FileTypeValidator.cls` | 0.5 days |
| FLS-enforced query patterns | All Apex controllers | 1 day (audit) |
| Security-focused test class | `force-app/main/default/classes/FileSecurityTest.cls` | 1 day |
| **Total** | | **3.5 days** |

---

## R6: Chat UI Performance -- Many Images

### Scenario

A chat thread contains 500+ messages, 50+ with images. When the LWC chat component loads, it must query messages, fetch ContentDocumentLink metadata for media messages, and render thumbnail URLs. Loading all images at once causes slow initial render and excessive SOQL consumption.

### Impact

- Slow initial load (3-10 seconds for 50+ images)
- SOQL limit pressure (100 queries in sync context)
- Browser memory consumption from simultaneous image loads
- Poor mobile experience

### SOLUTION: Lazy Loading + Pagination + Denormalized IDs + Virtual Scrolling

**1. Message Pagination (20 per page)**

```apex
@AuraEnabled(cacheable=true)
public static List<MessageWrapper> getMessages(Id chatId, Integer pageSize, String lastMessageId) {
    String query = 'SELECT Id, tgint__Text__c, tgint__Has_File__c, ' +
                   'tgint__Content_Version_Id__c, tgint__File_Type__c, ' +
                   'tgint__Content_Size__c, CreatedDate ' +
                   'FROM tgint__Messenger_Message__c ' +
                   'WHERE tgint__Chat__c = :chatId ';
    if (lastMessageId != null) {
        query += 'AND CreatedDate < (SELECT CreatedDate FROM tgint__Messenger_Message__c WHERE Id = :lastMessageId) ';
    }
    query += 'ORDER BY CreatedDate DESC LIMIT :pageSize';
    // ... execute with SECURITY_ENFORCED
}
```

**2. Denormalized Content Version ID**

Store `Content_Version_Id__c` directly on `Messenger_Message__c` or `Messenger_Attachment__c`. This eliminates the need for a separate ContentDocumentLink query:

```
Total SOQL per page load:
- 1 query: messages (paginated, 20 records)
- 0 queries: file metadata (denormalized on message record)
= 1 SOQL total (was 2 without denormalization)
```

**3. Lazy Image Loading in LWC**

```html
<template for:each={messages} for:item="msg">
    <template if:true={msg.hasFile}>
        <img
            src={msg.thumbnailUrl}
            alt={msg.fileName}
            loading="lazy"
            class="chat-thumbnail"
            data-message-id={msg.id}
            onerror={handleThumbnailError}
        />
    </template>
</template>
```

The `loading="lazy"` attribute defers image loading until the element enters the viewport.

**4. Thumbnail URL Construction (Zero Additional Queries)**

```javascript
get thumbnailUrl() {
    if (this.contentVersionId) {
        return `/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB240BY180&versionId=${this.contentVersionId}`;
    }
    return '/resource/tgint__placeholder_image'; // static resource fallback
}
```

**5. Virtual Scrolling for Long Threads**

For chats with 500+ messages, implement virtual scrolling that only renders DOM nodes for visible messages:

```javascript
// LWC virtual scroll: render only visible 20-message window
handleScroll(event) {
    const scrollTop = event.target.scrollTop;
    const visibleStart = Math.floor(scrollTop / this.messageHeight);
    const visibleEnd = visibleStart + this.visibleCount;
    this.renderedMessages = this.allMessages.slice(visibleStart, visibleEnd);
}
```

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| Paginated message query | `force-app/main/default/classes/ChatController.cls` | 1 day |
| Denormalized CV ID field | `force-app/main/default/objects/Messenger_Message__c/fields/` | 0.5 days |
| Lazy loading LWC | `force-app/main/default/lwc/chatMessage/` | 1 day |
| Virtual scrolling | `force-app/main/default/lwc/chatThread/` | 2 days |
| **Total** | | **4.5 days** |

---

## R7: Video Playback -- No Range Requests from SF Servlet

### Scenario

User sends a 200 MB video via Telegram. Video is stored as ContentVersion. When an agent clicks to play the video in the chat UI, the Salesforce file download servlet (`/sfc/servlet.shepherd/version/download/`) may or may not support HTTP `Range` headers. Evidence suggests it DOES work: the official [SalesforceLabs VideoViewer](https://github.com/SalesforceLabs/VideoViewer) AppExchange component provides "Mute, Seek, and Volume controls" using the same servlet URL. However, this has not been independently verified with explicit Range header testing. For large files, progressive download may still cause long initial load times.

### Impact

- 200 MB video at 50 Mbps: 32 seconds before playback starts (worst case; seeking may work via progressive download)
- 500 MB video: 80 seconds wait
- Seeking: likely works per SalesforceLabs VideoViewer evidence, but verify in scratch org
- Mobile: excessive data consumption, potential tab crashes on large files
- **Setup requirement:** File Upload and Download Security must set `.mp4`, `.webm`, `.mov` to "Execute in Browser" (not "Download")

### SOLUTION: Size-Tiered Playback Strategy

**Tier 1: Small Videos (< 25 MB) -- Auto-Play Inline**

```html
<!-- Videos under 25 MB: auto-play inline with download as progressive -->
<template if:true={isSmallVideo}>
    <video controls preload="auto" class="chat-video-inline"
           src={videoDownloadUrl} poster={videoPosterUrl}>
    </video>
</template>
```

At 50 Mbps, a 25 MB video downloads in 4 seconds -- acceptable for inline playback.

**Tier 2: Medium Videos (25-100 MB) -- Thumbnail + Click-to-Play**

```html
<!-- Videos 25-100 MB: show thumbnail, click triggers download+play -->
<template if:true={isMediumVideo}>
    <div class="chat-video-container" onclick={handleVideoPlay}>
        <img src={videoPosterUrl} class="chat-video-poster" />
        <div class="chat-video-overlay">
            <lightning-icon icon-name="utility:play" size="large"></lightning-icon>
            <span class="chat-video-size">{formattedFileSize}</span>
        </div>
    </div>
</template>
```

On click: show progress bar while downloading, then play.

**Tier 3: Large Videos (> 100 MB) -- Download Button Only**

```html
<!-- Videos over 100 MB: download button, no inline playback -->
<template if:true={isLargeVideo}>
    <div class="chat-video-download">
        <img src={videoPosterUrl} class="chat-video-poster-large" />
        <div class="chat-video-info">
            <span class="chat-video-title">{fileName}</span>
            <span class="chat-video-size">{formattedFileSize}</span>
            <lightning-button label="Download Video" onclick={handleVideoDownload}
                             icon-name="utility:download" variant="neutral">
            </lightning-button>
        </div>
    </div>
</template>
```

**Go-Side Video Thumbnail Generation**

For all videos, extract the first frame using ffmpeg before uploading to Salesforce:

```go
func extractVideoThumbnail(videoPath string) ([]byte, error) {
    cmd := exec.CommandContext(ctx, "ffmpeg",
        "-i", videoPath,
        "-ss", "00:00:01",      // 1 second in
        "-vframes", "1",         // 1 frame
        "-vf", "scale=480:-1",   // 480px wide, maintain aspect
        "-f", "image2pipe",
        "-vcodec", "mjpeg",
        "-",                     // output to stdout
    )
    return cmd.Output()
}
```

Upload thumbnail as a separate ContentVersion linked to the same message. Store thumbnail ContentVersionId on the attachment record.

**Salesforce NavigationMixin as Alternative**

For medium videos, open in Salesforce's native file preview modal which may handle video better:

```javascript
handleVideoPlay() {
    this[NavigationMixin.Navigate]({
        type: 'standard__namedPage',
        attributes: {
            pageName: 'filePreview'
        },
        state: {
            selectedRecordId: this.contentDocumentId
        }
    });
}
```

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| Size-tiered video component | `force-app/main/default/lwc/chatVideoPlayer/` | 2 days |
| Go ffmpeg thumbnail extraction | `internal/media/thumbnail.go` | 1 day |
| Thumbnail upload pipeline | `internal/media/uploader.go` (extend) | 0.5 days |
| Video poster field on attachment | `Messenger_Attachment__c.Thumbnail_Version_Id__c` | 0.5 days |
| **Total** | | **4 days** |

---

## R8: Managed Package Field Deprecation

### Scenario

In a 2GP managed package, custom fields and objects **cannot be deleted** in a package upgrade. If `Content_Version_Id__c`, `Content_Document_Id__c`, or `Has_File__c` are added now and later become unnecessary (e.g., switching storage backends), they remain as vestigial fields forever.

### Impact

- Schema bloat: stale fields visible in Object Manager
- Subscriber confusion: admins see unused fields
- No data loss or functional impact

### SOLUTION: Deprecation-in-Place Strategy

**1. Minimal Field Set**

Only add the absolute minimum fields needed:

| Field | Object | Type | Required? |
|---|---|---|---|
| `Has_File__c` | `Messenger_Message__c` | Checkbox | YES -- query optimization |
| `Content_Version_Id__c` | `Messenger_Attachment__c` | Text(18) | YES -- denormalized for performance |
| `File_Hash__c` | `Messenger_Attachment__c` | Text(64) | YES -- deduplication |
| `File_Type__c` | `Messenger_Attachment__c` | Text(10) | YES -- UI rendering |
| `Content_Size__c` | `Messenger_Attachment__c` | Number | YES -- size-tiered display |
| `Thumbnail_Version_Id__c` | `Messenger_Attachment__c` | Text(18) | OPTIONAL -- video thumbnails |

**2. If Deprecation Needed Later**

- Set field `Description` to `"DEPRECATED - No longer in use as of v2.x"`
- Remove from all page layouts and Lightning pages
- Remove from all SOQL queries in Apex (return null/blank)
- Keep field in schema but make it invisible to end users
- Document in release notes: "Field X is deprecated and will be ignored"

**3. Design for Dual Mode from Day One**

Implement a storage interface in Apex:

```apex
public interface IMediaStorage {
    String uploadFile(Blob fileData, String fileName, Id messageId);
    Blob downloadFile(String fileReference);
    void deleteFile(String fileReference);
}

public class SalesforceFilesStorage implements IMediaStorage {
    // ContentVersion-based implementation
}

// Future: public class R2Storage implements IMediaStorage { ... }
```

The `fileReference` is opaque -- could be a ContentVersionId or an R2 URL. The custom field `Media_Reference__c` stores whichever format is active.

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| IMediaStorage interface | `force-app/main/default/classes/IMediaStorage.cls` | 0.5 days |
| SalesforceFilesStorage impl | `force-app/main/default/classes/SalesforceFilesStorage.cls` | 1 day |
| Minimal field definitions | `force-app/main/default/objects/` | 0.5 days |
| **Total** | | **2 days** |

---

## R9: Multi-Org Storage Variance

### Scenario

MessageForge is installed across subscriber orgs with vastly different storage quotas. A Professional 10-user org has 16.1 GB total; an Enterprise 100-user org has 210 GB. A feature that works fine in one edition may fail in another.

### Math Model

| Edition | Base | Per User | 10-User Total | 100-User Total |
|---|---|---|---|---|
| Professional | 10 GB | 612 MB | 16.1 GB | 71.2 GB |
| Enterprise | 10 GB | 2 GB | 30 GB | 210 GB |
| Unlimited | 10 GB | 2 GB | 30 GB | 210 GB |
| Developer | 20 MB | N/A | 20 MB | 20 MB |

**Time to exhaust (200 media/day, 0.5 MB avg):**

| Edition (100 users) | Total Storage | Monthly Growth | Months to Exhaust |
|---|---|---|---|
| Professional | 71.2 GB | 3 GB | 24 months |
| Enterprise | 210 GB | 3 GB | 70 months |
| Unlimited | 210 GB | 3 GB | 70 months |

### Impact

- Professional edition orgs exhaust storage 3x faster than Enterprise
- Developer edition (20 MB) makes testing nearly impossible
- No one-size-fits-all configuration works across all editions

### SOLUTION: Runtime Storage Check + Edition-Aware Configuration + Graceful Degradation

**1. Runtime Storage Check via OrgLimits API**

```apex
public class StorageChecker {
    public static StorageInfo getStorageInfo() {
        Map<String, System.OrgLimit> limits = OrgLimits.getMap();
        System.OrgLimit fileStorage = limits.get('FileStorageMB');

        StorageInfo info = new StorageInfo();
        info.usedMB = fileStorage.getValue();
        info.totalMB = fileStorage.getLimit();
        info.percentUsed = (info.totalMB > 0)
            ? (Decimal.valueOf(info.usedMB) / info.totalMB * 100).setScale(1)
            : 0;
        info.messageForgeUsedMB = getMessageForgeUsage();
        return info;
    }

    private static Integer getMessageForgeUsage() {
        AggregateResult[] results = [
            SELECT SUM(ContentSize) totalSize FROM ContentVersion
            WHERE FirstPublishLocationId IN (
                SELECT Id FROM tgint__Messenger_Message__c
            )
            WITH SECURITY_ENFORCED
        ];
        Long totalBytes = (Long) results[0].get('totalSize');
        return (totalBytes != null) ? (Integer)(totalBytes / (1024 * 1024)) : 0;
    }
}
```

**2. Pre-Upload Storage Gate (Go side)**

Before every upload, check remaining storage via REST API:

```go
func (c *SFClient) checkStorageAvailable(ctx context.Context, fileSizeMB int) (bool, error) {
    resp, err := c.get(ctx, "/services/data/v62.0/limits")
    // Parse FileStorageMB from response
    used := resp.FileStorageMB.Value
    max := resp.FileStorageMB.Max
    remaining := max - used

    if remaining < fileSizeMB + c.storageBufferMB { // buffer = 100 MB
        return false, nil
    }
    return true, nil
}
```

**3. Admin Storage Dashboard**

LWC component on the MessageForge admin page showing:
- Org storage capacity and current usage (pie chart)
- MessageForge's share of total storage (highlighted segment)
- 30-day growth trend with projected exhaustion date
- Recommended actions based on edition:
  - Professional: "Consider Enterprise edition or enable retention policy"
  - Enterprise: "Storage healthy. Enable retention policy for long-term management."

**4. Graceful Degradation**

When storage is below threshold (configurable, default 100 MB remaining):
- Go middleware receives "storage low" signal from Apex (via REST response header or cached flag)
- Switch to metadata-only mode: store file metadata on message record but skip ContentVersion upload
- Chat UI shows: "Media not stored due to storage limits. View in Telegram."
- Admin notified via Platform Event

**5. Developer Edition Workaround**

For scratch org testing:
- Use mock ContentVersion service in test classes (no actual file storage)
- Scratch org definition: `"features": ["SalesforceFiles"]` to enable full Files functionality
- Test with tiny files (1 KB test images) to stay within 20 MB

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| StorageChecker Apex class | `force-app/main/default/classes/StorageChecker.cls` | 1 day |
| Go-side storage gate | `internal/salesforce/storage.go` | 0.5 days |
| Admin dashboard LWC | `force-app/main/default/lwc/storageHealth/` | 2 days |
| Graceful degradation logic | `internal/media/queue_worker.go` | 0.5 days |
| **Total** | | **4 days** |

---

## R10: GDPR File Deletion

### Scenario

A GDPR "right to erasure" request (Article 17) requires deleting all data for a specific Telegram user, including their media files stored as ContentVersion records. Deleting `Messenger_Message__c` does NOT automatically delete the linked ContentDocument.

### Impact

- ContentVersion records persist after message deletion (orphaned files)
- GDPR non-compliance: personal data retained after erasure request
- Storage leak: orphaned files consume quota indefinitely

### SOLUTION: Cascade Deletion Utility + Configurable Retention + Auto-Delete Trigger

**1. GDPR Cascade Deletion Utility**

```apex
public class GDPRDeletionService {

    public static DeletionResult deleteContactData(Id contactId) {
        DeletionResult result = new DeletionResult();

        // 1. Find all messages from this contact
        List<tgint__Messenger_Message__c> messages = [
            SELECT Id FROM tgint__Messenger_Message__c
            WHERE tgint__Sender_Contact__c = :contactId
            WITH SECURITY_ENFORCED
        ];
        Set<Id> messageIds = new Set<Id>();
        for (tgint__Messenger_Message__c m : messages) {
            messageIds.add(m.Id);
        }

        // 2. Find all ContentDocuments linked to these messages
        List<ContentDocumentLink> links = [
            SELECT ContentDocumentId FROM ContentDocumentLink
            WHERE LinkedEntityId IN :messageIds
            WITH SECURITY_ENFORCED
        ];
        Set<Id> docIds = new Set<Id>();
        for (ContentDocumentLink link : links) {
            docIds.add(link.ContentDocumentId);
        }

        // 3. Delete ContentDocuments (cascades to ContentVersion + ContentDocumentLink)
        if (Schema.sObjectType.ContentDocument.isDeletable() && !docIds.isEmpty()) {
            List<ContentDocument> docs = [SELECT Id FROM ContentDocument WHERE Id IN :docIds];
            delete docs;
            Database.emptyRecycleBin(docs); // Permanent deletion
            result.filesDeleted = docs.size();
        }

        // 4. Delete messages
        if (Schema.sObjectType.tgint__Messenger_Message__c.isDeletable()) {
            delete messages;
            Database.emptyRecycleBin(messages);
            result.messagesDeleted = messages.size();
        }

        // 5. Delete contact record
        if (Schema.sObjectType.tgint__Messenger_Contact__c.isDeletable()) {
            tgint__Messenger_Contact__c contact = [
                SELECT Id FROM tgint__Messenger_Contact__c WHERE Id = :contactId
            ];
            delete contact;
            Database.emptyRecycleBin(new List<SObject>{contact});
            result.contactDeleted = true;
        }

        return result;
    }
}
```

**2. Auto-Delete on Message Deletion (Trigger)**

```apex
trigger MessengerMessageTrigger on tgint__Messenger_Message__c (before delete) {
    // Get setting: should files be deleted with messages?
    tgint__Media_Settings__mdt settings = tgint__Media_Settings__mdt.getInstance('Default');

    if (settings != null && settings.tgint__Delete_Files_With_Messages__c) {
        Set<Id> messageIds = new Set<Id>();
        for (tgint__Messenger_Message__c msg : Trigger.old) {
            if (msg.tgint__Has_File__c) {
                messageIds.add(msg.Id);
            }
        }

        if (!messageIds.isEmpty()) {
            // Query and delete associated ContentDocuments
            List<ContentDocumentLink> links = [
                SELECT ContentDocumentId FROM ContentDocumentLink
                WHERE LinkedEntityId IN :messageIds
                WITH SECURITY_ENFORCED
            ];
            Set<Id> docIds = new Set<Id>();
            for (ContentDocumentLink link : links) {
                docIds.add(link.ContentDocumentId);
            }
            if (!docIds.isEmpty() && Schema.sObjectType.ContentDocument.isDeletable()) {
                delete [SELECT Id FROM ContentDocument WHERE Id IN :docIds];
            }
        }
    }
}
```

**3. Configurable Retention Policy**

Custom Metadata Type `tgint__Media_Settings__mdt`:
- `Delete_Files_With_Messages__c` (Boolean, default: true) -- cascade delete files when message is deleted
- `Retention_Days__c` (Number, default: 0 = keep forever) -- auto-delete after N days
- `Auto_Delete_Enabled__c` (Boolean, default: false) -- must be explicitly enabled by admin

**4. Orphan Cleanup Batch**

Scheduled batch job to find ContentDocuments created by MessageForge that are no longer linked to any active record:

```apex
global class OrphanFileCleanupBatch implements Database.Batchable<SObject>, Schedulable {
    global Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id FROM ContentDocument
            WHERE Id NOT IN (
                SELECT ContentDocumentId FROM ContentDocumentLink
                WHERE LinkedEntityId IN (SELECT Id FROM tgint__Messenger_Message__c)
            )
            AND CreatedBy.Profile.Name = 'MessageForge Integration'
        ]);
    }
    global void execute(Database.BatchableContext bc, List<ContentDocument> scope) {
        if (Schema.sObjectType.ContentDocument.isDeletable()) {
            delete scope;
            Database.emptyRecycleBin(scope);
        }
    }
}
```

### Implementation

| Component | File/Location | Effort |
|---|---|---|
| GDPRDeletionService | `force-app/main/default/classes/GDPRDeletionService.cls` | 1.5 days |
| Message deletion trigger | `force-app/main/default/triggers/MessengerMessageTrigger.trigger` | 1 day |
| Media_Settings__mdt | `force-app/main/default/customMetadata/` | 0.5 days |
| OrphanFileCleanupBatch | `force-app/main/default/classes/OrphanFileCleanupBatch.cls` | 1 day |
| **Total** | | **4 days** |

---

## Verified Limits Table

All values verified via WebSearch against Salesforce official documentation (Spring '26 / API v66.0) on 2026-03-30.

### File Storage Limits

| Limit | Value | Source | Workaround if Hit |
|---|---|---|---|
| Max file per ContentVersion (REST multipart) | 2 GB | [REST API Guide](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_sobject_insert_update_blob.htm) | Cap at configurable max (default 100 MB); metadata-only for oversized |
| REST API JSON body max | 50 MB | [REST API Guide](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_sobject_insert_update_blob.htm) | Always use multipart, never base64 for files > 37 MB |
| Base64 effective file limit | ~37.5 MB (before 33% encoding expansion) | Derived from 50 MB body limit | Use multipart form-data instead |
| REST API request timeout | 10 minutes (600,000ms) | [REST API Limits](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/resources_limits.htm) | Dynamic Go timeout: 30s base + 1min/100MB |
| ContentDocumentLink max per ContentDocument | 2,000 | [File Size and Sharing Limits](https://help.salesforce.com/s/articleView?id=experience.collab_files_size_limits.htm&language=en_US&type=5) | One link per message record; unlikely to hit |
| ContentDocumentLink bulk trigger support | NOT supported (fires once, not bulkified) | [Known Issue](https://help.salesforce.com/s/articleView?id=000384323&language=en_US&type=1) | Process one file at a time; use Queueable |
| Thumbnail auto-generation | Images, PDF/Office first page | [Salesforce Files docs](https://help.salesforce.com/s/articleView?id=experience.content_file_size_limits.htm&language=en_US&type=5) | Video: generate thumbnail with ffmpeg on Go side |
| Video thumbnail auto-generation | NOT supported | Salesforce Files documentation | Extract frame via ffmpeg, upload as separate ContentVersion |
| Rendition sizes | THUMB120BY90, THUMB240BY180, THUMB720BY480 | Salesforce Files documentation | Use THUMB240BY180 for chat bubbles |
| HTTP Range requests on download URL | LIKELY supported (SalesforceLabs VideoViewer has seeking) | [SalesforceLabs VideoViewer](https://github.com/SalesforceLabs/VideoViewer) | Size-tiered playback: auto-play < 25 MB, download > 100 MB |

### Storage Quotas by Edition

| Edition | Base File Storage | Per User License | 100-User Total | Workaround if Hit |
|---|---|---|---|---|
| Developer | 20 MB | N/A | 20 MB | Mock service in tests; tiny test files |
| Professional | 10 GB | 612 MB | 71.2 GB | Recommend Enterprise; enable retention policy |
| Enterprise | 10 GB | 2 GB | 210 GB | Retention policy + compression; purchase add-on storage |
| Unlimited | 10 GB | 2 GB | 210 GB | Retention policy for long-term management |
| Additional storage | $5/GB/month (~$60/GB/year) | Purchasable | -- | Customer purchases as needed |

Sources: [Salesforce Files Storage Allocations](https://help.salesforce.com/s/articleView?id=experience.files_storage.htm&language=en_US&type=5), [CloudFiles Storage Cost Guide](https://www.cloudfiles.io/blog/salesforce-file-storage-cost), [XfilesPro Analysis](https://www.xfilespro.com/detailed-analysis-of-salesforce-file-storage-cost/)

### API Call Limits (24-hour rolling)

| Edition | API Calls / 24h | Calculation | Workaround if Hit |
|---|---|---|---|
| Professional | 15,000 | Fixed | Queue uploads; spread over 24h |
| Enterprise (100 users) | 100,000 | max(100,000; users x 1,000) | Rate limiter + Connected App 5x increase |
| Unlimited | Effectively uncapped | max(unlimited; users x 5,000) | N/A |
| Connected App ISV increase | Up to 5x base | ISV partner request | Apply during Partner enrollment |
| API call add-on | +10,000/day per $25/month | Self-service SKU | Customer purchases as needed |

Sources: [API Limits Blog](https://developer.salesforce.com/blogs/2024/11/api-limits-and-monitoring-your-api-usage), [API Limits Reference](https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_api.htm), [Enterprise Allocations](https://help.salesforce.com/s/articleView?id=xcloud.overview_limits_enterprise.htm&language=en_US&type=5)

### Governor Limits (Per Apex Transaction)

| Limit | Synchronous | Asynchronous | Workaround if Hit |
|---|---|---|---|
| SOQL queries | 100 | 200 | Use Queueable for file ops (async = 200) |
| DML statements | 150 | 150 | Bulk insert ContentVersion records |
| Records retrieved by SOQL | 50,000 | 50,000 | Paginate queries; never query all files |
| Heap size | 6 MB | 12 MB | Never query VersionData in lists; process files on Go side |
| CPU time | 10,000 ms | 60,000 ms | Async processing for bulk operations |
| Callouts (HTTP) | 100 | 100 | Batch callouts; use Platform Events |
| Callout timeout | 120 sec | 120 sec | N/A (file upload is inbound, not Apex callout) |

Sources: [Apex Heap Size](https://help.salesforce.com/s/articleView?id=000384468&language=en_US&type=1), [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_class_System_OrgLimits.htm), [OrgLimits Class](https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_class_System_OrgLimits.htm)

### ContentVersion-Specific Constraints

| Constraint | Value | Impact | Workaround |
|---|---|---|---|
| VersionData blob in Apex heap | Limited by 6/12 MB heap | Cannot process large files in triggers | Process on Go side; Apex handles metadata only |
| Composite API blob support | NOT supported | Each file = 1 API call | Go-side queue + rate limiter |
| Bulk API 2.0 blob support | NOT supported | Cannot batch file uploads | Queue + rate limit; spread over time |
| Platform Event blob support | NOT supported (1 MB payload) | Cannot send files via PE | REST API direct upload from Go |
| Storage per version | Full file size per version | Version history multiplies storage | Disable versioning or auto-delete old versions |

---

## Total Implementation Effort

| Risk | Solution Effort |
|---|---|
| R1: Storage Growth | 5 days |
| R2: API Call Budget | 5 days |
| R3: Large Video Upload | 3.5 days |
| R4: Trigger Conflicts | 2.5 days |
| R5: AppExchange Security | 3.5 days |
| R6: Chat UI Performance | 4.5 days |
| R7: Video Playback | 4 days |
| R8: Field Deprecation | 2 days |
| R9: Multi-Org Variance | 4 days |
| R10: GDPR Deletion | 4 days |
| **Total (sequential)** | **38 days** |
| **Total (parallelized Go + Apex)** | **~22 days** |

Go-side work (R1 compression, R2 queue, R3 streaming, R7 thumbnails) and Apex-side work (R4 async, R5 security, R6 UI, R9 dashboard, R10 deletion) can proceed in parallel.

---

---

## Web Research Sources

All limits verified via WebSearch on 2026-03-30 against official Salesforce documentation:

- [REST API - Insert or Update Blob Data](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_sobject_insert_update_blob.htm) -- 2 GB multipart max, 50 MB JSON body max
- [REST API Limits endpoint](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/resources_limits.htm) -- 10-minute request timeout, runtime limits query
- [API Limits and Monitoring](https://developer.salesforce.com/blogs/2024/11/api-limits-and-monitoring-your-api-usage) -- Per-edition API allocations, add-on pricing ($25/month per 10K/day)
- [API Request Limits and Allocations](https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_api.htm) -- Enterprise 1,000/user/day, Unlimited 5,000/user/day
- [Enterprise Edition Allocations](https://help.salesforce.com/s/articleView?id=xcloud.overview_limits_enterprise.htm&language=en_US&type=5) -- Edition-specific limits
- [Files Storage Allocations](https://help.salesforce.com/s/articleView?id=experience.files_storage.htm&language=en_US&type=5) -- Storage per edition, per-user allocations
- [File Size and Sharing Limits](https://help.salesforce.com/s/articleView?id=experience.collab_files_size_limits.htm&language=en_US&type=5) -- ContentDocumentLink limits
- [STORAGE_LIMIT_EXCEEDED Error](https://help.salesforce.com/s/articleView?id=000380367&language=en_US&type=1) -- Grace period ~110%, error behavior
- [XfilesPro - Storage Limit Exceeded](https://www.xfilespro.com/salesforce-file-storage-limit-exceeded-some-use-cases-tips-to-prevent-hitting-storage-limits/) -- Storage exceeded behavior, monitoring strategies
- [OrgLimits Class](https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_class_System_OrgLimits.htm) -- `OrgLimits.getMap().get('FileStorageMB')` for runtime monitoring
- [InfallibleTechie - OrgLimits for Storage](https://www.infallibletechie.com/2023/07/how-to-find-data-storage-and-file-storage-using-salesforce-apex.html) -- Apex code example for storage checks
- [Apex Heap Size Error](https://help.salesforce.com/s/articleView?id=000384468&language=en_US&type=1) -- 6 MB sync / 12 MB async heap limits
- [ContentDocument Trigger Behavior](https://help.salesforce.com/s/articleView?id=000381623&language=en_US&type=1) -- Trigger limitations on ContentDocumentLink
- [ContentDocumentLink Bulk Issue](https://salesforcetech.blog/2025/03/24/beware-contentdocument-and-contentdocumentlink-triggers-does-not-work-in-bulk-mode/) -- Triggers don't fire in bulk mode
- [SalesforceLabs VideoViewer](https://github.com/SalesforceLabs/VideoViewer) -- Video playback with seeking works in Salesforce
- [AppExchange Security Review Guide](https://noltic.com/stories/how-to-pass-salesforce-appexchange-security-review) -- FLS is #1 rejection reason
- [Salesforce Code Analyzer for AppExchange](https://developer.salesforce.com/docs/platform/salesforce-code-analyzer/guide/appexchange.html) -- Required scan rules
- [Remove Components from Managed Packages](https://developer.salesforce.com/docs/atlas.en-us.pkg1_dev.meta/pkg1_dev/packaging_managed_component_deletion.htm) -- Fields cannot be deleted from managed packages
- [Deprecate Managed Apex](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_manpkgs_deprecated.htm) -- @Deprecated annotation for Apex classes
- [GDPR Right to Be Forgotten](https://help.salesforce.com/s/articleView?id=sf.right_to_be_forgotten.htm&language=en_US&type=5) -- Salesforce GDPR deletion policies
- [LWC Lazy Loading Guide](https://sfdcdevelopers.com/2025/09/25/guide-to-lwc-lazy-loading/) -- IntersectionObserver, dynamic imports, conditional rendering
- [CloudFiles Storage Cost Guide](https://www.cloudfiles.io/blog/salesforce-file-storage-cost) -- Additional storage $5/GB/month
- [SMS-Magic Storage Migration](https://www.sms-magic.com/docs/salesforce/knowledge-base/send-a-multimedia-message/) -- Competitor migrated from external to Salesforce Files
- [ContentVersion Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/api/sforce_api_objects_contentversion.htm) -- FirstPublishLocationId, field specifications

*Analysis completed 2026-03-30. All limits verified via WebSearch against official Salesforce documentation.*
