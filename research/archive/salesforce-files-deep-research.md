# Salesforce Files Deep Web Research

> Verified research with live web sources. Extends `salesforce-contentdocument-media-storage.md`.
> Date: 2026-03-30

---

## Topic 1: Competitor Analysis -- How AppExchange Messaging Apps Store Files

### Findings

**SMS-Magic (Conversive)**
- SMS-Magic historically stored media files on their own external servers ("SMS-Magic storage"). This caused problems: files were lost in conversations and not associated with records.
- They **migrated to Salesforce Files storage** (ContentVersion). Users can now configure: (a) incoming media stored in Salesforce local storage, (b) outgoing media stored in Salesforce local storage, (c) both directions stored in Salesforce, (d) files associated with the primary object record.
- This migration was driven by data privacy concerns and the need for larger file sizes.
- Source: [SMS-Magic Documentation](https://www.sms-magic.com/docs/salesforce/knowledge-base/send-a-multimedia-message/), [SMS-Magic AppExchange](https://appexchange.salesforce.com/appxListingDetail?listingId=a0N300000024XvyEAE)

**Mogli SMS & WhatsApp**
- Mogli is built natively on Salesforce (5-star rated, Salesforce AppExchange).
- File attachments for MMS use Salesforce's native ContentDocument architecture. They integrate with Salesforce Files for media handling.
- Source: [Mogli AppExchange](https://appexchange.salesforce.com/appxListingDetail?listingId=a0N3A00000DqCytUAF)

**Heymarket**
- Heymarket syncs messages between their platform and Salesforce every minute.
- Their Salesforce integration works with ContentDocument -- Zapier triggers fire when "an attachment, note, or content document is added or updated."
- SOC 2 Type 2, HIPAA, and TCPA-compliant.
- Source: [Heymarket Salesforce Integration](https://help.heymarket.com/hc/en-us/articles/360034372271-Salesforce-Integration), [Heymarket AppExchange](https://appexchange.salesforce.com/appxListingDetail?listingId=a0N3A00000FMaZuUAL)

**Salesforce Digital Engagement (1st party)**
- Salesforce's own Digital Engagement add-on for Service Cloud uses ContentVersion for file attachments in messaging channels (SMS, WhatsApp, Facebook Messenger).
- The Spring '24 release added file attachment support for Messaging for In-App and Web, using Salesforce Files.
- Source: [Salesforce Digital Engagement Guide](https://help.salesforce.com/s/articleView?id=000389088&language=en_US&type=1), [File Attachments Release Notes](https://help.salesforce.com/s/articleView?id=release-notes.rn_miaw_file_attachments.htm&language=en_US&release=240&type=5)

**360 SMS App**
- Sends single/MMS/bulk messages. File handling follows the standard Salesforce ContentVersion pattern.
- Source: [360 SMS AppExchange](https://appexchange.salesforce.com/appxListingDetail?listingId=a0N3000000DpSyIEAV)

**Sinch Engage (formerly Mercury SMS)**
- Send and automate SMS, MMS, WhatsApp, and RCS inside Salesforce.
- Every message kept on record using Salesforce-native storage.
- Source: [Sinch Engage AppExchange](https://appexchange.salesforce.com/appxListingDetail?listingId=a0N3000000B4DyvEAF)

### Summary Table

| App | File Storage | External Storage? | Passed Security Review |
|---|---|---|---|
| SMS-Magic | Salesforce Files (migrated from external) | No (migrated away) | Yes |
| Mogli | Salesforce Files (native) | No | Yes |
| Heymarket | Salesforce Files (synced) | Heymarket servers + sync to SF | Yes |
| SF Digital Engagement | Salesforce Files (native) | No | N/A (1st party) |
| 360 SMS | Salesforce Files (native) | No | Yes |
| Sinch Engage | Salesforce Files (native) | No | Yes |

### Confidence: HIGH
Industry consensus is clear: **every major messaging AppExchange app stores media in Salesforce Files (ContentVersion)**. SMS-Magic's migration away from external storage is particularly telling -- they moved TO Salesforce Files for better data association and privacy.

---

## Topic 2: ContentVersion REST API -- Verified Details

### Findings

**Maximum file size:**
- ContentVersion objects: **2 GB** maximum file size.
- Non-multipart (JSON body with Base64): 50 MB text data or **37.5 MB** before Base64 encoding expansion.
- Multipart upload: **2 GB** (supported since API v56.0+, Winter '23).
- Source: [Salesforce REST API - Insert or Update Blob Data](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_sobject_insert_update_blob.htm)

**Multipart request format (verified):**
- First part: JSON metadata (Title, PathOnClient, FirstPublishLocationId, Description, etc.)
- Second part: Binary file data with Content-Type header.
- Content-Disposition for metadata: `form-data; name="entity_content"`
- Content-Disposition for blob: `form-data; name="VersionData"; filename="<name>"`
- Source: [Salesforce REST API Developer Guide v66.0](https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/api_rest.pdf)

**Required fields for ContentVersion insert:**
- `Title` -- file display name
- `PathOnClient` -- filename with extension (determines FileType)
- `VersionData` -- the binary blob (multipart) or Base64 string (JSON body)
- Source: [Salesforce ContentVersion Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_contentversion.htm)

**FirstPublishLocationId with namespaced custom objects:**
- **Confirmed: works with any record ID.** FirstPublishLocationId is a polymorphic reference accepting object, user, or Library (ContentWorkspace) IDs. Custom object records (including namespaced like `tgint__Messenger_Message__c`) are accepted because the link is by record ID, not API name.
- Setting FirstPublishLocationId auto-creates a ContentDocumentLink, saving a DML operation.
- **Caveat:** When uploading a second version of an existing ContentDocument, FirstPublishLocationId is left blank on the new ContentVersion.
- Source: [ContentVersion Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_contentversion.htm), [Known Issue](https://issues.salesforce.com/issue/a028c00000gAye9AAC/firstpublishlocationid-on-contentversion-changed-upon-insert-of-second-version-via-api)

**Batching limitations (confirmed):**
- Composite API does NOT support blob/binary data for ContentVersion.
- Bulk API 2.0 does NOT support ContentVersion blob uploads.
- Each file = 1 REST API call. No workaround.
- Source: [Salesforce REST API Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_sobject_insert_update_blob.htm)

### Confidence: HIGH
All claims verified against official Salesforce developer documentation (Spring '26 / v66.0).

---

## Topic 3: Storage Pricing & What Happens When Full

### Findings

**File storage allocation per edition:**

| Edition | Base (org-wide) | Per User License | Example: 100 users |
|---|---|---|---|
| Professional | 10 GB | 612 MB | 10 + 59.8 = ~70 GB |
| Enterprise | 10 GB | 2 GB | 10 + 200 = **210 GB** |
| Unlimited | 10 GB | 2 GB | 10 + 200 = **210 GB** |
| Performance | 10 GB | 2 GB | 10 + 200 = **210 GB** |

- Source: [Salesforce Files Storage Allocations](https://help.salesforce.com/s/articleView?id=experience.files_storage.htm&language=en_US&type=5), [CloudFiles Storage Cost Guide](https://www.cloudfiles.io/blog/salesforce-file-storage-cost), [XfilesPro Analysis](https://www.xfilespro.com/detailed-analysis-of-salesforce-file-storage-cost/)

**Additional storage pricing:**
- **$5 per GB per month** (~$60/GB/year) for additional file storage.
- Can be purchased through Salesforce account team.
- Source: [CloudFiles Storage Cost Guide](https://www.cloudfiles.io/blog/salesforce-file-storage-cost), [XfilesPro Cost Analysis](https://www.xfilespro.com/detailed-analysis-of-salesforce-file-storage-cost/)

**What happens when storage is exceeded:**
1. **Grace period up to ~110% of allocated storage.** Salesforce does not immediately block at 100%.
2. Beyond 110%, **uploads are blocked** and file creation fails. Automated processes (Flows, triggers, API inserts) that create files will fail.
3. Error message: `STORAGE_LIMIT_EXCEEDED`
4. Salesforce sends admin notifications when approaching limits.
5. Both data storage and file storage must be under 100% to avoid errors.
- Source: [XfilesPro - File Storage Limit Exceeded](https://www.xfilespro.com/salesforce-file-storage-limit-exceeded-some-use-cases-tips-to-prevent-hitting-storage-limits/), [Salesforce Help - Storage Limit Error](https://help.salesforce.com/s/articleView?id=000380367&language=en_US&type=1), [Connecting Software - Storage Limit](https://www.connecting-software.com/blog/salesforce-storage-limit-exceeded/)

**What counts against file storage:**
- ContentVersion file data (VersionData blob)
- Each version of a file counts separately
- Salesforce-generated thumbnails/renditions do NOT count
- Source: [Salesforce Files Storage Allocations](https://help.salesforce.com/s/articleView?id=experience.files_storage.htm&language=en_US&type=5)

### Calculation: 100-user Enterprise Org

- Base: 10 GB
- Per-user: 100 x 2 GB = 200 GB
- **Total: 210 GB**
- At $5/GB/month for additional: 100 GB extra = $500/month

### Confidence: HIGH
Storage allocation numbers confirmed across multiple sources including official Salesforce help docs.

### Solutions for Storage Management
1. **Monitor proactively:** Use Setup > Storage Usage page. Set up alerts at 70%, 85%, 95%.
2. **Lifecycle management:** Implement retention policies -- archive/delete media from old closed conversations.
3. **Purchase additional storage** if needed ($5/GB/month).
4. **Compress before upload:** On Go side, resize images and transcode video before uploading to Salesforce.
5. **Customer communication:** Document expected storage impact in package description on AppExchange so customers can plan.

---

## Topic 4: Video -- Solving the Streaming Problem

### Findings

**The concern:** Salesforce's `/sfc/servlet.shepherd/` download servlet may not support HTTP Range requests, which are needed for HTML5 `<video>` seeking (jumping to a specific timestamp).

**Evidence that video playback DOES work:**

1. **SalesforceLabs VideoViewer** -- An official Salesforce Labs LWC component that plays video files natively in Salesforce. Supports mp4, mov, webm, m4v. Provides "Mute, Seek, and Volume controls." This is an AppExchange-listed component.
   - Source: [SalesforceLabs/VideoViewer](https://github.com/SalesforceLabs/VideoViewer)

2. **HTML5 `<video>` with servlet.shepherd URL works.** Multiple community sources confirm that using `/sfc/servlet.shepherd/document/download/{ContentDocumentId}` as the `src` attribute of an HTML5 `<video>` element allows playback with seeking.
   - Source: [forcePanda - Play Audio and Video Files](https://forcepanda.wordpress.com/2020/06/23/how-to-play-audio-and-videos-files-in-salesforce/), [forcePanda - Preview or Play Video](https://forcepanda.wordpress.com/2020/07/02/preview-or-play-video-and-audio-files-in-salesforce-without-any-extra-customizations/)

3. **File Upload and Download Security setting:** For video playback in browser, the `.mp4` (and other video) file type must be set to "Execute in Browser" instead of "Download" in Setup > File Upload and Download Security.
   - Source: [forcePanda](https://forcepanda.wordpress.com/2020/07/02/preview-or-play-video-and-audio-files-in-salesforce-without-any-extra-customizations/)

4. **Progressive download is the mechanism.** HTML5 browsers support seeking to not-yet-downloaded portions of video using HTTP 1.1 range requests. The browser handles this natively when the server supports it.
   - Source: [Harbinger Group - Video Streaming in HTML5](https://www.harbingergroup.com/blogs/video-streaming-in-html5-video-tag/)

**URL patterns for video:**
```
// Full download (works as video src)
/sfc/servlet.shepherd/document/download/{ContentDocumentId}
/sfc/servlet.shepherd/version/download/{ContentVersionId}
```

**Setup requirement:**
- CSP Trusted Sites: Add your org's own domain for self-referencing URLs
- File Upload and Download Security: Set video file types to "Execute in Browser"

### Confidence: MED
SalesforceLabs VideoViewer confirms seeking works. Multiple community posts confirm HTML5 video with servlet.shepherd URLs works. However, no official Salesforce documentation explicitly confirms HTTP Range request support on the servlet. The evidence is strong but indirect.

### Solutions for Video Challenges

| Challenge | Solution |
|---|---|
| No seeking support (IF servlet doesn't support Range) | Use SalesforceLabs VideoViewer pattern -- it works with seeking controls |
| Large video files slow to load | Generate thumbnail on Go side (ffmpeg, extract first frame), display thumbnail, click to play |
| No auto-generated video thumbnails | Upload thumbnail as separate ContentVersion, store reference on Messenger_Attachment__c |
| Very large videos (>100 MB) | Show download link instead of inline player; user downloads and plays locally |
| Browser compatibility | mp4 (H.264) is universally supported; transcode other formats on Go side before upload |

---

## Topic 5: API Call Budget -- Real Numbers

### Findings

**API limit formula (verified):**

```
Total Daily Limit = 100,000 base + (num_licenses x per_license_allocation) + purchased add-ons
```

**Per-license allocations by edition:**

| Edition | Salesforce License | Platform License | Community License |
|---|---|---|---|
| Enterprise | 1,000/user/day | 1,000/user/day | 200/user/day |
| Unlimited | 5,000/user/day | 5,000/user/day | 200/user/day |
| Performance | 5,000/user/day | 5,000/user/day | 200/user/day |

- Source: [Salesforce API Limits Blog](https://developer.salesforce.com/blogs/2024/11/api-limits-and-monitoring-your-api-usage), [Coupler.io Salesforce API Limits](https://blog.coupler.io/salesforce-api-limits/), [Salesforce Developer Limits Reference](https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_api.htm)

**Example org calculations:**

| Scenario | Enterprise (50 users) | Unlimited (50 users) |
|---|---|---|
| Base | 100,000 | 100,000 |
| Per-user | 50 x 1,000 = 50,000 | 50 x 5,000 = 250,000 |
| **Total daily limit** | **150,000** | **350,000** |

**Media upload modeling:**

| Volume | API calls/day | % of Enterprise (50u) | % of Unlimited (50u) |
|---|---|---|---|
| 50 media/day (light) | ~75 (50 uploads + 25 downloads) | 0.05% | 0.02% |
| 500 media/day (moderate) | ~750 | 0.5% | 0.2% |
| 5,000 media/day (heavy) | ~7,500 | 5.0% | 2.1% |
| 50,000 media/day (extreme) | ~75,000 | 50% | 21.4% |

Note: Each media file requires 1 API call for upload. Downloads from Go also cost 1 API call each. Estimated download ratio: 50% of uploads (not every file is re-downloaded for outbound).

**Purchasing additional API calls:**
- **$25/month per 10,000 additional API calls per day.**
- Can be purchased via the "My Account" app in the org as a self-service SKU: "Additional API Calls - 10,000 per day."
- Source: [Salesforce API Limits Blog](https://developer.salesforce.com/blogs/2024/11/api-limits-and-monitoring-your-api-usage)

**Emergency temporary increase:**
- Contact Salesforce Support / Account Executive.
- Maximum 2-week temporary increase.
- Intended for data migrations, not ongoing use.
- Source: [Salesforce Help - Temporary API Limit Increase](https://help.salesforce.com/s/articleView?id=000382675&language=en_US&type=1)

### Confidence: HIGH
API limit formulas verified against official Salesforce developer blog and limits documentation.

### Solutions for High API Usage

| Strategy | Impact |
|---|---|
| **Upgrade to Unlimited edition** | 5x per-user allocation (5,000 vs 1,000) |
| **Purchase API add-on packs** | $25/month per 10K/day; 100K extra = $250/month |
| **Queue and throttle uploads on Go side** | Smooth burst traffic; use Go channel with rate limiter |
| **Skip duplicate downloads** | Cache ContentVersionId-to-file mapping in PostgreSQL; don't re-download known files |
| **Batch metadata operations** | Use Composite API for non-blob operations (creating ContentDocumentLinks, updating message records) to reduce calls |
| **Monitor usage** | Check `/services/data/vXX.0/limits` endpoint; alert at 70% daily consumption |

---

## Topic 6: AppExchange Security Review -- ContentVersion Requirements

### Findings

**CRUD/FLS enforcement (mandatory):**
- Before any CRUD operation on ContentVersion, ContentDocumentLink, or ContentDocument, check user permissions.
- Use `WITH SECURITY_ENFORCED` in SOQL queries.
- Use `Schema.sObjectType.ContentVersion.isCreateable()` before DML inserts.
- Use `Security.stripInaccessible(AccessType.READABLE, records)` for query results.
- Failure to enforce CRUD/FLS is the **#1 reason apps fail security review.**
- Source: [Noltic - AppExchange Security Review Guide](https://noltic.com/stories/how-to-pass-salesforce-appexchange-security-review), [Salesforce ISVforce Guide](https://developer.salesforce.com/docs/atlas.en-us.packagingGuide.meta/packagingGuide/security_review_guidelines.htm)

**Required tooling for submission:**
- Salesforce Code Analyzer (mandatory) with `--rule-selector AppExchange --rule-selector Recommended:Security`
- Checkmarx scan
- ZAP or Burp Suite DAST for external APIs (i.e., the Go middleware endpoints)
- Note: Chimera DAST Scanner retired June 2025; replaced by other scanners.
- Source: [Salesforce Code Analyzer for AppExchange](https://developer.salesforce.com/docs/platform/salesforce-code-analyzer/guide/appexchange.html), [Noltic](https://noltic.com/stories/how-to-pass-salesforce-appexchange-security-review)

**File type validation:**
- Validate MIME type and file extension before creating ContentVersion.
- Block executable file types (.exe, .bat, .sh, .js) unless explicitly needed.
- Salesforce has built-in file type security (Setup > File Upload and Download Security), but the package should add its own validation layer.
- Source: Community best practices; no single official source for this specific requirement.

**Sharing model for ContentDocumentLink:**
- Use `ShareType = 'I'` (Inferred) so access follows the linked record's sharing rules.
- Use `Visibility = 'AllUsers'` for internal org use, or `'InternalUsers'` to exclude community/portal users.
- Security reviewers check that file visibility is appropriate for the use case.
- Source: [ContentDocumentLink Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_contentdocumentlink.htm)

**Trigger on ContentDocumentLink:**
- Managed packages CAN include triggers on standard objects including ContentDocumentLink.
- One trigger per object per namespace (no conflict with other packages).
- Limitation: `addError()` method does NOT work in ContentDocumentLink triggers for showing custom error messages to users during file uploads.
- Source: [Salesforce Help - ContentDocument Triggers](https://help.salesforce.com/s/articleView?id=000381623&language=en_US&type=1), [SalesforceCodeCrack - CDL Trigger](https://www.salesforcecodecrack.com/2019/02/salesforce-contentdocumentlink-trigger.html)

**Permission Sets:**
- Package must include a Permission Set granting: Read + Create on ContentVersion, Read + Create on ContentDocumentLink, Read on ContentDocument.
- ContentDocument access is implicitly granted through ContentDocumentLink sharing.
- Source: AppExchange best practices (multiple sources)

**Documented bypass cases:**
- If you bypass CRUD/FLS for system operations (e.g., creating log records, system metadata), document these cases in the security review submission.
- Source: [Salesforce Developer Blog - AppExchange Security](https://developer.salesforce.com/blogs/2023/04/prepare-your-app-to-pass-the-appexchange-security-review)

**Review timeline:**
- Submission verification: 1-2 weeks
- Technical testing (penetration testing): 3-6 weeks
- Total: 4-8 weeks
- Source: [Noltic](https://noltic.com/stories/how-to-pass-salesforce-appexchange-security-review)

### Confidence: HIGH
Requirements verified against official ISVforce Guide, Salesforce Code Analyzer docs, and multiple experienced ISV sources.

---

## Topic 7: Performance at Scale -- Real-World Reports

### Findings

**ContentDocumentLink query restrictions (verified):**
- SOQL on ContentDocumentLink **requires** a filter on `ContentDocumentId` or `LinkedEntityId` using `=` or `IN` operators.
- Cannot use LIKE, wildcard, or unfiltered queries.
- Cannot query only on SystemModStamp (affects incremental backup strategies).
- Source: [ContentDocumentLink Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_contentdocumentlink.htm), [Salesforce Release Notes - Query All File Shares](https://help.salesforce.com/s/articleView?id=release-notes.rn_experiences_files_query_all.htm&language=en_US&release=246&type=5)

**ContentDocumentLink does NOT support bulkification well:**
- Official known issue: ContentDocumentLink doesn't support bulkification.
- Source: [Salesforce Help - ContentDocumentLink bulkification](https://help.salesforce.com/s/articleView?id=000384323&language=en_US&type=1)

**Query pattern recommendations for scale:**
- Always filter by `LinkedEntityId` (indexed) -- this is the most efficient query pattern.
- Use subqueries when needed: `SELECT ... FROM ContentDocumentLink WHERE LinkedEntityId IN (SELECT Id FROM ...)`
- For large data volumes: process asynchronously in batches using Batch Apex (up to 50 million records).
- Source: [Trailhead - Large Data Volumes](https://trailhead.salesforce.com/content/learn/modules/large-data-volumes/conduct-data-queries-and-searches)

**Denormalization strategy (essential for chat UI):**
- Store `Has_File__c` (Boolean) and `File_Count__c` (Number) on `Messenger_Message__c`.
- Only query ContentDocumentLink for messages where `Has_File__c = true`.
- Store `Content_Version_Id__c` on `Messenger_Attachment__c` to construct display URLs without SOQL.
- This avoids N+1 query patterns in chat feeds.

**ContentVersion limits:**
- Maximum ContentVersions per document: configurable, default varies by org. Can be increased via Salesforce Support.
- No hard limit on total ContentVersion records per org.
- Source: [Salesforce Help - ContentVersion Limits](https://help.salesforce.com/s/articleView?id=000386874&language=en_US&type=1)

**Maximum ContentDocumentLinks per ContentDocument:**
- The existing research document cites 2,000 max links per ContentDocument. This could not be independently verified in official documentation during this research. However, for our use case (typically 1-2 links per file: message record + optionally chat record), this is irrelevant.
- Source: Not confirmed with URL; treat as UNVERIFIED.

**SOQL governor limits apply normally:**
- 100 SOQL queries per synchronous transaction
- 50,000 records retrieved per transaction
- Standard indexing on LinkedEntityId ensures fast queries when filtered properly

### Confidence: MED
Query restrictions are well-documented. Performance at true enterprise scale (millions of ContentDocumentLink records) is less well-documented publicly -- most sources focus on general Salesforce large data volume patterns rather than ContentDocumentLink specifically. However, the denormalization strategy (Has_File__c + Content_Version_Id__c) effectively eliminates the query performance concern for our chat UI use case.

### Solutions for Scale Challenges

| Challenge | Solution |
|---|---|
| Slow chat feed with many media messages | Denormalize: `Has_File__c` boolean skips file queries for text-only messages |
| N+1 SOQL queries for files | Store `Content_Version_Id__c` directly on attachment record for URL construction |
| Bulk query for all files in a chat | Query by chat record ID with single SOQL: `WHERE LinkedEntityId = :chatId` |
| ContentDocumentLink query restrictions | Always filter by `LinkedEntityId` (indexed); never run unfiltered queries |
| Very old conversations with many files | Pagination: query messages first, then files only for visible messages |
| Trigger processing at scale | ContentDocumentLink trigger checks `LinkedEntityId` prefix to filter only Messenger_Message records; skip all others |

---

## Unresolved Questions (Need Scratch Org Testing)

1. **HTTP Range request support:** Does `/sfc/servlet.shepherd/version/download/` return `Accept-Ranges: bytes` header? Test with: `curl -I -H "Authorization: Bearer <token>" "https://<instance>/sfc/servlet.shepherd/version/download/<ContentVersionId>"`. If yes, HTML5 video seeking works natively. If no, the SalesforceLabs VideoViewer pattern is the fallback (and it demonstrably works with seeking).

2. **FirstPublishLocationId with namespaced custom objects via REST API:** While confirmed to work in general, test specifically with `tgint__Messenger_Message__c` record IDs via the Go middleware REST API upload to verify no namespace-related issues.

3. **ContentDocumentLink trigger firing on REST API uploads:** Verify that when Go middleware uploads a ContentVersion with FirstPublishLocationId, the auto-created ContentDocumentLink fires the `after insert` trigger correctly.

4. **File Upload and Download Security defaults:** Check whether `.mp4`, `.webm`, `.mov` are set to "Execute in Browser" or "Download" by default in a fresh scratch org. If "Download", document the post-install setup step.

5. **Storage consumption measurement:** Upload 100 test files of various sizes (1KB to 50MB) and verify that Salesforce's reported file storage consumption matches the sum of file sizes (and that thumbnails/renditions truly don't count).

6. **Concurrent upload throughput:** Benchmark how many parallel ContentVersion REST API uploads the Salesforce org can sustain before hitting rate limits or experiencing degraded performance. Test with 10, 50, 100 concurrent uploads.

7. **Content_Version_Id__c on insert:** When using FirstPublishLocationId, the ContentVersionId is returned in the REST response. Verify the exact response format and whether the ContentDocumentId is also returned (or requires a follow-up query).

8. **Lightning file preview modal for video:** Test whether the NavigationMixin `filePreview` page correctly handles video files and provides a playback UI, or only shows a download button.

---

## Summary of Key Facts (Quick Reference)

| Fact | Value | Confidence |
|---|---|---|
| Max file size (multipart upload) | 2 GB | HIGH |
| Max file size (Base64 JSON body) | ~37.5 MB | HIGH |
| API call per file upload | 1 | HIGH |
| API call per file download | 1 | HIGH |
| Enterprise API limit (50 users) | 150,000/day | HIGH |
| Unlimited API limit (50 users) | 350,000/day | HIGH |
| Additional API calls cost | $25/month per 10K/day | HIGH |
| File storage per Enterprise user | 2 GB/user + 10 GB base | HIGH |
| Additional file storage cost | $5/GB/month | HIGH |
| Storage exceeded grace period | Up to ~110% | MED |
| Video playback with seeking | Works (SalesforceLabs VideoViewer confirms) | MED |
| Competitors using ContentVersion | All major (SMS-Magic, Mogli, Heymarket, Sinch) | HIGH |
| CRUD/FLS required for security review | Yes (mandatory, #1 failure reason) | HIGH |
| ContentDocumentLink query requires filter | Yes (LinkedEntityId or ContentDocumentId) | HIGH |
| FirstPublishLocationId with custom objects | Works (polymorphic) | HIGH |
| Security review timeline | 4-8 weeks | MED |
