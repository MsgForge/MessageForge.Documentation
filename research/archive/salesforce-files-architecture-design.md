# ContentVersion Media Storage — Complete Architecture Design

**Date:** 2026-03-30
**Status:** ACCEPTED
**ADR:** [ADR-20](../architecture/adr-20-media-storage-pivot.md)
**Supersedes:** ADR-3 (R2 + Workers), ADR-14 (SigV4 presigned URLs)
**Web Research Verification:** [salesforce-files-deep-research.md](./salesforce-files-deep-research.md)

---

## Table of Contents

1. [Updated System Architecture](#1-updated-system-architecture)
2. [Complete Data Model Changes](#2-complete-data-model-changes)
3. [Inbound Flow Design (Telegram -> SF)](#3-inbound-flow-design)
4. [Outbound Flow Design (SF -> Telegram)](#4-outbound-flow-design)
5. [LWC Display Design](#5-lwc-display-design)
6. [SOQL Query Design](#6-soql-query-design)
7. [Components Added/Removed/Modified](#7-components-addedremoved-modified)
8. [ContentDocumentLink Trigger Design](#8-contentdocumentlink-trigger-design)

---

## 1. Updated System Architecture

### 1A. Inbound Data Flow (Telegram -> Salesforce)

```
Telegram Server
    |
    |  MTProto messages.getMedia / Bot API getFile
    v
Go Middleware
    |  1. Download binary from Telegram
    |  2. Generate thumbnail (ffmpeg for video; WebP->PNG for sticker)
    |  3. Create Messenger_Message__c via REST API (with Contains_Inbound_Files__c=true)
    |  4. POST multipart/form-data to Salesforce REST API:
    |     POST /services/data/v62.0/sobjects/ContentVersion
    |     (ContentVersion with FirstPublishLocationId = Message record ID)
    v
Salesforce REST API
    |  5. ContentVersion created (VersionData = binary)
    |  6. ContentDocument auto-created (parent container)
    |  7. ContentDocumentLink auto-created -> Messenger_Message__c
    v
ContentDocumentLink Trigger (tgint__ContentDocumentLinkTrigger)
    |  8. Detects LinkedEntityId is a Messenger_Message__c (by key prefix)
    |  9. Updates Has_File__c = true, File_Count__c on the Message
    v
Go Middleware (continued)
    |  10. Query back ContentDocumentId:
    |      GET /services/data/v62.0/sobjects/ContentVersion/{id}?fields=ContentDocumentId
    |  11. Upload thumbnail as separate ContentVersion (video only)
    |  12. Create Messenger_Attachment__c with Content_Version_Id__c,
    |      Content_Document_Id__c, Thumbnail_CV_Id__c
    v
LWC (same-origin /sfc/servlet.shepherd/)
    |  13. Renders thumbnail inline in chat bubble
    |  14. Click opens NavigationMixin filePreview (standard Salesforce modal)
```

### 1B. Outbound Data Flow (Salesforce -> Telegram)

```
LWC Chat UI
    |  1. Agent uses <lightning-file-upload> to attach file
    |     (record-id = draft Messenger_Message__c ID)
    |     Component auto-creates ContentVersion + ContentDocumentLink
    v
ContentDocumentLink Trigger
    |  2. Fires after insert -> Has_File__c = true on draft message
    v
Apex Controller (sendOutboundMessage @AuraEnabled)
    |  3. Agent clicks Send -> LWC calls Apex with chatId + text + fileDocumentIds
    |  4. Creates Messenger_Message__c (Direction__c = 'outbound', Has_File__c = true)
    |  5. Queries ContentVersion details (Id, ContentDocumentId, Title, FileType, ContentSize)
    |  6. Creates Messenger_Attachment__c per file with CV/CD IDs
    |  7. Enqueues OutboundMediaQueueable (Queueable Apex for async callout)
    v
OutboundMediaQueueable (Apex -> Go REST callout)
    |  8. POST to Go: { messageId, contentVersionId, contentDocumentId, fileName, mimeType }
    v
Go Middleware
    |  9. GET /services/data/v62.0/sobjects/ContentVersion/{id}/VersionData
    |     (streams binary via io.Copy, no full buffering)
    |  10. Pipes directly to Telegram upload:
    |      MTProto: messages.sendMedia (InputMediaUploadedDocument/Photo)
    |      Bot API: sendPhoto / sendDocument / sendVideo (multipart)
    v
Go Middleware (delivery confirmation)
    |  11. Publishes tgint__Message_Delivery_Status__e Platform Event
    |      (Status__c = DELIVERED | FAILED, Message_External_ID__c, Timestamp__c)
    v
Dual Consumer Pattern
    |  12. LWC empApi -> real-time UI update (green checkmark)
    |  13. Apex Trigger -> persists to Messenger_Message__c.Delivery_Status__c
```

### 1C. Revised System Diagram

```
                    +------------------+
                    |   Messaging      |
                    |   Platforms      |
                    |   (Telegram,     |
                    |   WhatsApp, ...) |
                    +--------+---------+
                             |
                    +--------v---------+         +-----------------+
                    |     Go           |<------->|   PostgreSQL    |
                    |   Middleware     |         |   (sessions,    |
                    +--+------+--------+         |    queues)      |
                       |      |                  +-----------------+
              +--------+      +--------+
              |                        |
      +-------v------+         +------v----------+
      |  Salesforce   |         |   Centrifugo    |
      |  REST API     |         |   (WebSocket)   |
      +-------+-------+         +------+----------+
              |                        |
      +-------v-------+               |
      | ContentVersion |               |
      | (file binary)  |               |
      +-------+-------+               |
              |                        |
      +-------v-------+               |
      | ContentDocument|               |
      | Link -> Message|               |
      +-------+-------+               |
              |                        |
      +-------v-----------------------v+
      |           LWC (Browser)         |
      |  /sfc/servlet.shepherd/...      |
      |  (same-origin, no CORS needed)  |
      +---------------------------------+
```

**Key architectural change:** Cloudflare R2, Workers, CDN proxy, presigned URLs, and CORS/CSP
configuration for external media domains are all eliminated. Media is served from the same
Salesforce origin as the LWC, eliminating all cross-origin complexity.

---

## 2. Complete Data Model Changes

### 2A. New Fields on `Messenger_Message__c`

| API Name | Type | Length | Default | Description |
|---|---|---|---|---|
| `tgint__Has_File__c` | Checkbox | -- | `false` | Fast filter: does this message have attached files? Set by ContentDocumentLink trigger. Never set directly by Go or Apex application code. |
| `tgint__Contains_Inbound_Files__c` | Checkbox | -- | `false` | Marks inbound messages with media. Set by Go during message creation. Distinct from `Has_File__c` because it is direction-aware and set before file upload completes. Useful for analytics (e.g., "how many photos did customers send"). |
| `tgint__File_Count__c` | Number(3,0) | -- | `0` | Denormalized count of ContentDocumentLinks to this message. Updated by trigger on insert/delete. Enables UI badge (e.g., "3 files") and album layout logic without additional SOQL. |

### 2B. New Fields on `Messenger_Attachment__c`

| API Name | Type | Length | Default | Description |
|---|---|---|---|---|
| `tgint__Content_Version_Id__c` | Text | 18 | -- | Stores the latest ContentVersion ID (068xxx). Enables direct URL construction in LWC: `/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB720BY480&versionId={this_field}`. Eliminates need to query ContentDocumentLink. |
| `tgint__Content_Document_Id__c` | Text | 18 | -- | Stores the ContentDocument ID (069xxx). Used for NavigationMixin filePreview (`selectedRecordId`), file deletion, and linking multiple records. |
| `tgint__Thumbnail_CV_Id__c` | Text | 18 | -- | ContentVersion ID of a separate thumbnail image. Used for video (Go-generated via ffmpeg first frame) and animated stickers (Go-rendered first frame). Null for photos -- Salesforce auto-generates renditions for image types (THUMB120BY90, THUMB240BY180, THUMB720BY480). |
| `tgint__Waveform_JSON__c` | Text | 1000 | -- | JSON array of waveform amplitude values for voice messages (e.g., `[5,12,28,31,15,...]`). Extracted by Go from Telegram voice message metadata (128 bars, 5-bit values). Used by LWC to render visual waveform bars. Only populated when `Message_Type__c = 'voice'`. |

### 2C. Deprecation Strategy for Existing URL Fields

In a 2GP managed package, fields **cannot be deleted** after the first managed release.
These fields must be deprecated, not removed.

| Field | Strategy | Details |
|---|---|---|
| `tgint__Messenger_Message__c.Media_URL__c` | **Deprecate** | Set `Description` to `"DEPRECATED v2.0 -- replaced by ContentVersion (ADR-20). Do not use."`. Stop writing to it in new code. Remove from all page layouts and list views. Set field-level security to hidden for all profiles/permission sets. Keep field definition for upgrade safety -- existing subscriber orgs may have automation referencing it. |
| `tgint__Messenger_Message__c.Media_MIME_Type__c` | **Keep** | Still useful. MIME type drives LWC rendering decisions (photo vs video vs voice vs document). The data source changes (Telegram metadata instead of R2 metadata) but the field purpose is identical. |
| `tgint__Messenger_Attachment__c.Media_URL__c` | **Deprecate** | Same deprecation pattern as above. |
| `tgint__Messenger_Attachment__c.Thumbnail_URL__c` | **Deprecate** | Replaced by `Thumbnail_CV_Id__c`. Same deprecation pattern. |

**Migration path for existing data:** If any subscriber org has data in `Media_URL__c` fields (pointing to R2 from a pre-v2.0 install), a one-time post-upgrade Batch Apex job can be provided that: (a) downloads from R2, (b) creates ContentVersion records, and (c) populates the new CV/CD ID fields. This is an optional post-upgrade script, not automatic. For v1.0 launch (no existing subscribers), this is not needed.

---

## 3. Inbound Flow Design (Telegram -> SF)

### Step-by-Step with Exact API Calls

**Step 1: Go receives media from Telegram**

```
MTProto: messages.getMedia -> upload.getFile (streaming, 512KB chunks, up to 2 GB)
Bot API: GET https://api.telegram.org/file/bot<token>/<file_path> (up to 20 MB)
```

Go holds the binary in a streaming reader. For files under 50 MB, buffering in memory is
acceptable. For files over 50 MB (MTProto only), stream to a temp file on disk.

**Step 2: Go generates thumbnail (video/sticker only)**

For video messages, Go extracts the first frame using ffmpeg:

```bash
ffmpeg -i input.mp4 -vframes 1 -f image2 -vcodec mjpeg -q:v 5 pipe:1
```

This produces a JPEG thumbnail (~10-50 KB). For stickers:
- Static stickers (WebP): Go converts to PNG via image/webp decoder.
- Animated stickers (TGS/Lottie): Go renders the first frame to PNG.
- Video stickers (WebM): Go extracts first frame via ffmpeg.

Photos do NOT need thumbnails -- Salesforce auto-generates renditions for image file types
(THUMB120BY90, THUMB240BY180, THUMB720BY480).

**Step 3: Go creates Messenger_Message__c record**

```http
POST /services/data/v62.0/sobjects/tgint__Messenger_Message__c
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "tgint__Chat__c": "<Chat record ID>",
  "tgint__Message_Text__c": "<caption if any>",
  "tgint__Direction__c": "inbound",
  "tgint__Message_Type__c": "photo",
  "tgint__Sender_Name__c": "John Doe",
  "tgint__Sender_External_ID__c": "123456789",
  "tgint__Message_External_ID__c": "tg:12345:67890",
  "tgint__Contains_Inbound_Files__c": true,
  "tgint__Sent_At__c": "2026-03-30T14:23:05.000Z"
}
```

Response returns the `Messenger_Message__c` record ID (e.g., `a0B7Q00000XXXXX`).

**Step 4: Go uploads file to Salesforce ContentVersion**

```http
POST /services/data/v62.0/sobjects/ContentVersion
Content-Type: multipart/form-data; boundary=------MessageForge
Authorization: Bearer <access_token>

------MessageForge
Content-Disposition: form-data; name="entity_content"
Content-Type: application/json

{
  "Title": "photo_2026-03-30_142305.jpg",
  "PathOnClient": "photo_2026-03-30_142305.jpg",
  "FirstPublishLocationId": "a0B7Q00000XXXXX",
  "Description": "tg:chat=12345:msg=67890"
}
------MessageForge
Content-Disposition: form-data; name="VersionData"; filename="photo_2026-03-30_142305.jpg"
Content-Type: image/jpeg

<binary file data>
------MessageForge--
```

**Critical:** `FirstPublishLocationId` is set to the `Messenger_Message__c` record ID.
This causes Salesforce to auto-create a `ContentDocumentLink` between the new ContentDocument
and the Message record. This saves a separate DML operation and is atomic. Verified: works
with namespaced custom object record IDs because the link is by record ID, not API name.

**Verified multipart format** (from Salesforce REST API Developer Guide):
- First part: `name="entity_content"` with `Content-Type: application/json` containing metadata fields.
- Second part: `name="VersionData"` with `filename="<name>"` and appropriate `Content-Type` containing binary data.
- Maximum file size via multipart: **2 GB** (supported since API v56.0+, Winter '23).
- Maximum via Base64 JSON body: **~37.5 MB** before encoding (50 MB after Base64 expansion).
- **Composite API does NOT support blob/binary data.** Each file = 1 REST API call. No batching.
- **Bulk API 2.0 does NOT support ContentVersion blob uploads.**

**Response:**

```json
{
  "id": "068XXXXXXXXXXXX",
  "success": true,
  "errors": []
}
```

The `id` is the ContentVersion ID.

**Step 5: ContentDocumentLink trigger fires (automatic)**

The `ContentDocumentLink` auto-created in Step 4 fires the `tgint__ContentDocumentLinkTrigger`.
The trigger:
1. Detects the `LinkedEntityId` is a `Messenger_Message__c` record (by 3-character key prefix).
2. Runs aggregate COUNT query on `ContentDocumentLink WHERE LinkedEntityId IN :affectedIds`.
3. Sets `Has_File__c = true` and `File_Count__c = <count>`.

This is automatic and requires no action from Go.

**Step 6: Go queries back ContentDocumentId**

```http
GET /services/data/v62.0/sobjects/ContentVersion/068XXXXXXXXXXXX?fields=ContentDocumentId
Authorization: Bearer <access_token>
```

Response:

```json
{
  "ContentDocumentId": "069XXXXXXXXXXXX",
  "Id": "068XXXXXXXXXXXX"
}
```

**Optimization for album messages:** Batch this query for multiple ContentVersions:

```http
GET /services/data/v62.0/query?q=SELECT+Id,ContentDocumentId+FROM+ContentVersion+WHERE+Id+IN+('068XXX','068YYY','068ZZZ')
```

This consumes 1 SOQL-based API call instead of N separate field lookups.

**Step 7: Go uploads thumbnail (video/sticker only)**

If the message is a video or animated sticker, Go uploads the extracted thumbnail as a
separate ContentVersion with `FirstPublishLocationId` set to the same Message record:

```http
POST /services/data/v62.0/sobjects/ContentVersion
(same multipart format, with the JPEG/PNG thumbnail binary, ~10-50 KB)
```

This produces a second ContentVersion ID and a second ContentDocumentLink to the same message.

**Step 8: Go creates Messenger_Attachment__c record**

```http
POST /services/data/v62.0/sobjects/tgint__Messenger_Attachment__c
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "tgint__Message__c": "a0B7Q00000XXXXX",
  "tgint__Content_Version_Id__c": "068XXXXXXXXXXXX",
  "tgint__Content_Document_Id__c": "069XXXXXXXXXXXX",
  "tgint__Thumbnail_CV_Id__c": "068YYYYYYYYYY",
  "tgint__Media_MIME_Type__c": "video/mp4",
  "tgint__File_Name__c": "video_2026-03-30.mp4",
  "tgint__File_Size__c": 15728640,
  "tgint__Sort_Order__c": 1,
  "tgint__Duration_Seconds__c": 45,
  "tgint__Width__c": 1920,
  "tgint__Height__c": 1080,
  "tgint__Media_External_ID__c": "AgACAgIAAxkBAAI..."
}
```

For voice messages, Go also populates `tgint__Waveform_JSON__c` with the Telegram waveform data.

### Album Messages (Multiple Photos)

For Telegram album messages (mediaGroupId), Go:

1. Creates one `Messenger_Message__c` with `Message_Type__c = 'album'` and `Contains_Inbound_Files__c = true`.
2. Uploads each photo as a separate ContentVersion with `FirstPublishLocationId` = Message ID.
3. Creates one `Messenger_Attachment__c` per photo with `Sort_Order__c` = 1, 2, 3...
4. The ContentDocumentLink trigger fires for each link, and the aggregate re-count ensures `File_Count__c` is correct regardless of execution order.

### API Call Budget Per Inbound Media Message

| Operation | API Calls | Notes |
|---|---|---|
| Create Messenger_Message__c | 1 | JSON POST (can be part of Platform Event flow instead) |
| Upload ContentVersion (file) | 1 | Multipart POST. No batching possible for binary. |
| Query ContentDocumentId | 1 | Can batch for albums (1 SOQL query for N files) |
| Upload thumbnail CV (video only) | 0-1 | Only for video/sticker |
| Create Messenger_Attachment__c | 1 | JSON POST |
| **Total (photo)** | **4** | |
| **Total (video)** | **5** | Includes thumbnail upload |
| **Total (album of 10)** | **14** | 1 msg + 10 uploads + 1 batch query + 1 batch attachment create (Composite) |

**Optimization:** Messenger_Attachment__c creation can use Composite API (up to 25 subrequests)
to batch multiple attachment records in a single HTTP call. This saves N-1 API calls for albums.

**Rate limiting:** Go maintains a per-org rate limiter (default: 50 media uploads/minute =
72,000/day). Configurable per customer. When API budget drops below 20% remaining (checked via
`/services/data/v62.0/limits/`), uploads queue in PostgreSQL and drain during off-peak hours.

---

## 4. Outbound Flow Design (SF -> Telegram)

### Step-by-Step

**Step 1: Agent attaches file in LWC**

The chat compose area includes a `<lightning-file-upload>` component:

```html
<lightning-file-upload
    label="Attach"
    name="outboundFile"
    accept=".jpg,.jpeg,.png,.gif,.mp4,.webm,.pdf,.doc,.docx,.xls,.xlsx,.zip"
    record-id={draftMessageRecordId}
    onuploadfinished={handleUploadFinished}
    multiple
></lightning-file-upload>
```

**Verified capabilities** (from Salesforce Component Library):
- `record-id`: Associates uploaded files with a record via ContentDocumentLink. If omitted, file is private to the uploading user.
- `multiple`: Allows up to 10 files simultaneously (default Salesforce limit).
- Drag-and-drop supported.
- Automatically creates ContentVersion + ContentDocumentLink (zero custom upload code).
- `onuploadfinished` event returns `{ files: [{ documentId, name }] }` per file.
- For guest users, returns ContentVersionId instead of ContentDocumentId.

**Step 2: ContentVersion created automatically by the component**

When the agent selects a file, `lightning-file-upload` automatically:
1. Creates a ContentVersion record with the binary data.
2. Creates a ContentDocumentLink to the `record-id` (the draft Message record).
3. The ContentDocumentLink trigger fires -> sets `Has_File__c = true` on the message.

**Step 3: LWC captures file metadata**

```javascript
handleUploadFinished(event) {
    const uploadedFiles = event.detail.files;
    this.pendingFiles = uploadedFiles.map(file => ({
        documentId: file.documentId,
        name: file.name
    }));
}
```

**Step 4: Agent clicks Send -> LWC calls Apex**

```javascript
async handleSend() {
    const result = await sendOutboundMessage({
        chatId: this.chatId,
        text: this.messageText,
        fileDocumentIds: this.pendingFiles.map(f => f.documentId)
    });
    // Clear compose area
    this.messageText = '';
    this.pendingFiles = [];
}
```

**Step 5: Apex creates records and notifies Go**

```apex
@AuraEnabled
public static String sendOutboundMessage(Id chatId, String text, List<Id> fileDocumentIds) {
    // 1. Create Messenger_Message__c (outbound)
    Messenger_Message__c msg = new Messenger_Message__c(
        Chat__c = chatId,
        Message_Text__c = text,
        Direction__c = 'outbound',
        Message_Type__c = fileDocumentIds.isEmpty() ? 'text' : 'document',
        Delivery_Status__c = 'pending',
        Has_File__c = !fileDocumentIds.isEmpty(),
        File_Count__c = fileDocumentIds.size()
    );
    insert msg;

    if (!fileDocumentIds.isEmpty()) {
        // 2. Query ContentVersion details (CRUD/FLS enforced)
        List<ContentVersion> versions = [
            SELECT Id, ContentDocumentId, Title, FileType, ContentSize
            FROM ContentVersion
            WHERE ContentDocumentId IN :fileDocumentIds
              AND IsLatest = true
            WITH SECURITY_ENFORCED
        ];

        // 3. Create Messenger_Attachment__c per file
        List<Messenger_Attachment__c> attachments = new List<Messenger_Attachment__c>();
        Integer sortOrder = 1;
        for (ContentVersion cv : versions) {
            attachments.add(new Messenger_Attachment__c(
                Message__c = msg.Id,
                Content_Version_Id__c = cv.Id,
                Content_Document_Id__c = cv.ContentDocumentId,
                File_Name__c = cv.Title,
                Media_MIME_Type__c = mapFileTypeToMime(cv.FileType),
                File_Size__c = cv.ContentSize,
                Sort_Order__c = sortOrder++
            ));
        }
        insert attachments;

        // 4. Async callout to Go middleware via Queueable
        System.enqueueJob(new OutboundMediaQueueable(msg.Id, versions));
    }

    return msg.Id;
}
```

**Why Queueable (not @future or Platform Event):**
- Queueable supports retry via chaining (`System.enqueueJob` from within execute).
- Queueable supports passing complex types (List<ContentVersion>).
- Queueable is monitored in Apex Jobs UI.
- Platform Events would work but add unnecessary indirection for a 1:1 callout.

**Step 6: Go downloads file from Salesforce**

Go receives the callback with ContentVersion IDs and downloads each file via streaming:

```http
GET /services/data/v62.0/sobjects/ContentVersion/068XXXXXXXXXXXX/VersionData
Authorization: Bearer <access_token>
```

Response: raw binary stream with `Content-Type` header matching the file type.
Go pipes directly to Telegram upload without full buffering:

```go
sfResp, err := sfClient.Do(sfReq)
if err != nil { return fmt.Errorf("download contentversion: %w", err) }
defer sfResp.Body.Close()

// Stream directly to Telegram — zero copy in Go memory
err = tgUploader.SendMedia(ctx, chatID, sfResp.Body, sfResp.ContentLength, fileName, mimeType)
```

**Step 7: Go sends to Telegram**

```
MTProto: messages.sendMedia (InputMediaUploadedDocument/Photo)
  - Files > 10 MB: upload.saveBigFilePart with 512 KB chunks, streaming from SF response body
Bot API: sendPhoto / sendDocument / sendVideo (multipart upload, max 50 MB)
```

**Step 8: Delivery status flows back**

Go publishes `tgint__Message_Delivery_Status__e` Platform Event:
- `Status__c` = `DELIVERED` / `FAILED` / `RETRYING`
- `Message_External_ID__c` = Telegram message ID
- `Error_Code__c` + `Error_Detail__c` (on failure)
- `Timestamp__c` = delivery timestamp

The dual-consumer pattern (LWC empApi + Apex Trigger) updates the UI in real-time and
persists the status to `Messenger_Message__c.Delivery_Status__c`.

---

## 5. LWC Display Design

### 5A. Photo Display

**Thumbnail in chat bubble:**

```html
<template if:true={isPhoto}>
    <div class="chat-photo-container" onclick={handlePhotoClick}>
        <img
            src={photoThumbnailUrl}
            alt={attachment.File_Name__c}
            class="chat-photo-thumb"
            loading="lazy"
        />
    </div>
</template>
```

**URL pattern (verified via Salesforce documentation and community sources):**

```javascript
get photoThumbnailUrl() {
    // THUMB720BY480 for chat bubble display (high quality, reasonable size)
    return `/sfc/servlet.shepherd/version/renditionDownload`
        + `?rendition=THUMB720BY480`
        + `&versionId=${this.attachment.Content_Version_Id__c}`;
}
```

Available rendition sizes:
- `THUMB120BY90` -- 120x90, for chat list previews and small thumbnails
- `THUMB240BY180` -- 240x180, for grid views
- `THUMB720BY480` -- 720x480, for chat bubble display (recommended default)

These renditions are **auto-generated by Salesforce** for image and PDF file types.
They do NOT count against file storage.

**Click-to-preview (full-size) using NavigationMixin:**

```javascript
import { NavigationMixin } from 'lightning/navigation';

handlePhotoClick() {
    this[NavigationMixin.Navigate]({
        type: 'standard__namedPage',
        attributes: { pageName: 'filePreview' },
        state: {
            recordIds: this.allPhotoDocumentIds.join(','),
            selectedRecordId: this.attachment.Content_Document_Id__c
        }
    });
}
```

**Verified behavior** (from Salesforce LWC Developer Guide):
- `filePreview` named page supports ContentDocument and ContentHubItem objects.
- `recordIds`: comma-separated list of ALL ContentDocument IDs available for navigation.
- `selectedRecordId`: the specific file to show first.
- Opens Salesforce's native file preview modal with: full-resolution image, zoom in/out,
  navigation arrows when multiple files (album), and download button.
- Supported in Lightning Experience and Salesforce mobile app.
- NOT supported in Experience Cloud sites (use direct download URL as fallback).

**Album grid layout (multiple photos):**

```html
<template if:true={isAlbum}>
    <div class={albumGridClass}>
        <template for:each={attachments} for:item="att">
            <div key={att.Id} class="album-cell" onclick={handleAlbumPhotoClick}
                 data-document-id={att.Content_Document_Id__c}>
                <img
                    src={att.thumbnailUrl}
                    alt={att.File_Name__c}
                    class="album-thumb"
                    loading="lazy"
                />
            </div>
        </template>
    </div>
</template>
```

Grid layout CSS classes:
- 1 photo: full width (max-width: 320px)
- 2 photos: 2-column, equal width
- 3 photos: 1 left (tall) + 2 stacked right
- 4+ photos: 2x2 grid with "+N" overlay on last cell if more than 4

### 5B. Video Display

**Chat bubble with poster image and play button overlay:**

```html
<template if:true={isVideo}>
    <div class="chat-video-container" onclick={handleVideoClick}>
        <template if:true={videoThumbnailUrl}>
            <img
                src={videoThumbnailUrl}
                alt="Video"
                class="video-poster"
                loading="lazy"
            />
        </template>
        <template if:false={videoThumbnailUrl}>
            <div class="video-placeholder">
                <lightning-icon icon-name="doctype:video" size="large"></lightning-icon>
            </div>
        </template>
        <div class="video-play-overlay">
            <lightning-icon icon-name="utility:play" size="large"></lightning-icon>
        </div>
        <span class="video-duration">{formattedDuration}</span>
    </div>
</template>
```

**Thumbnail URL** uses the Go-generated thumbnail ContentVersion (Salesforce does NOT
auto-generate thumbnails for video file types):

```javascript
get videoThumbnailUrl() {
    if (this.attachment.Thumbnail_CV_Id__c) {
        return `/sfc/servlet.shepherd/version/renditionDownload`
            + `?rendition=THUMB720BY480`
            + `&versionId=${this.attachment.Thumbnail_CV_Id__c}`;
    }
    return null; // Falls back to SLDS doctype:video icon
}
```

**Click behavior -- size-based strategy:**

```javascript
handleVideoClick() {
    const fileSizeBytes = this.attachment.File_Size__c;
    const ONE_HUNDRED_MB = 100 * 1024 * 1024;

    if (fileSizeBytes > ONE_HUNDRED_MB) {
        // Large videos: trigger download. User plays locally.
        window.open(
            `/sfc/servlet.shepherd/version/download/${this.attachment.Content_Version_Id__c}`,
            '_blank'
        );
    } else {
        // Smaller videos: inline HTML5 player in modal
        this.showVideoModal = true;
        this.videoSourceUrl =
            `/sfc/servlet.shepherd/version/download/${this.attachment.Content_Version_Id__c}`;
    }
}
```

**Inline video modal (for files under 100 MB):**

```html
<template if:true={showVideoModal}>
    <section class="slds-modal slds-fade-in-open slds-modal_large" role="dialog">
        <div class="slds-modal__container">
            <header class="slds-modal__header">
                <button class="slds-button slds-modal__close" onclick={closeVideoModal}>
                    <lightning-icon icon-name="utility:close" size="small"></lightning-icon>
                </button>
                <h2 class="slds-text-heading_medium">{attachment.File_Name__c}</h2>
            </header>
            <div class="slds-modal__content slds-p-around_none">
                <video controls autoplay class="video-player-full">
                    <source src={videoSourceUrl} type={videoMimeType} />
                </video>
            </div>
        </div>
    </section>
    <div class="slds-backdrop slds-backdrop_open" onclick={closeVideoModal}></div>
</template>
```

**Video playback mechanism:** The `/sfc/servlet.shepherd/version/download/` URL serves the
full binary file. The browser's `<video>` element handles progressive download and buffering.
Evidence from SalesforceLabs VideoViewer (official Salesforce Labs component on AppExchange)
confirms that seeking controls work with servlet.shepherd URLs. For most chat videos
(under 50 MB from Bot API), this provides acceptable playback UX.

**Setup requirement:** In Setup > File Upload and Download Security, `.mp4`, `.webm`, and
`.mov` file types must be set to "Execute in Browser" (not "Download") for inline playback
to work. Document this as a post-install configuration step.

### 5C. Voice Message Display

**Chat bubble with audio player and waveform:**

```html
<template if:true={isVoice}>
    <div class="chat-voice-container">
        <div class="voice-waveform">
            <template for:each={waveformBars} for:item="bar">
                <div key={bar.key} class="waveform-bar" style={bar.style}></div>
            </template>
        </div>
        <audio controls class="voice-player">
            <source
                src={voiceDownloadUrl}
                type={voiceMimeType}
            />
        </audio>
        <span class="voice-duration">{formattedDuration}</span>
    </div>
</template>
```

**URL:**

```javascript
get voiceDownloadUrl() {
    return `/sfc/servlet.shepherd/version/download/${this.attachment.Content_Version_Id__c}`;
}
```

**Waveform visualization approach:**

Telegram voice messages include a `waveform` byte array in the message metadata (128 bars,
5-bit values). Go extracts this during inbound processing and stores it as a JSON array on
`tgint__Waveform_JSON__c`:

```javascript
get waveformBars() {
    if (!this.attachment.Waveform_JSON__c) return [];
    const values = JSON.parse(this.attachment.Waveform_JSON__c);
    const maxVal = Math.max(...values);
    return values.map((v, i) => ({
        key: `bar-${i}`,
        height: maxVal > 0 ? (v / maxVal) * 100 : 0,
        style: `height: ${maxVal > 0 ? (v / maxVal) * 100 : 0}%`
    }));
}
```

```css
.voice-waveform {
    display: flex;
    align-items: flex-end;
    gap: 1px;
    height: 32px;
    padding: 4px 0;
}
.waveform-bar {
    width: 2px;
    min-height: 2px;
    background-color: var(--lwc-colorBrand, #0176d3);
    border-radius: 1px;
    transition: background-color 0.1s;
}
```

### 5D. Document Display

**Chat bubble with file type icon and metadata:**

```html
<template if:true={isDocument}>
    <div class="chat-document-container" onclick={handleDocumentClick}>
        <div class="document-icon">
            <lightning-icon
                icon-name={documentIconName}
                size="medium"
            ></lightning-icon>
        </div>
        <div class="document-info">
            <span class="document-name">{attachment.File_Name__c}</span>
            <span class="document-size">{formattedFileSize}</span>
        </div>
        <div class="document-actions">
            <lightning-button-icon
                icon-name="utility:download"
                alternative-text="Download"
                onclick={handleDocumentDownload}
            ></lightning-button-icon>
        </div>
    </div>
</template>
```

**SLDS doctype icon mapping:**

```javascript
get documentIconName() {
    const mimeToIcon = {
        'application/pdf': 'doctype:pdf',
        'application/msword': 'doctype:word',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document': 'doctype:word',
        'application/vnd.ms-excel': 'doctype:excel',
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet': 'doctype:excel',
        'application/vnd.ms-powerpoint': 'doctype:ppt',
        'application/vnd.openxmlformats-officedocument.presentationml.presentation': 'doctype:ppt',
        'application/zip': 'doctype:zip',
        'application/x-rar-compressed': 'doctype:zip',
        'text/plain': 'doctype:txt',
        'text/csv': 'doctype:csv',
        'text/html': 'doctype:html',
        'application/json': 'doctype:xml'
    };
    return mimeToIcon[this.attachment.Media_MIME_Type__c] || 'doctype:attachment';
}
```

**Click-to-preview (NavigationMixin filePreview):**

```javascript
handleDocumentClick(event) {
    event.stopPropagation();
    this[NavigationMixin.Navigate]({
        type: 'standard__namedPage',
        attributes: { pageName: 'filePreview' },
        state: {
            recordIds: this.attachment.Content_Document_Id__c,
            selectedRecordId: this.attachment.Content_Document_Id__c
        }
    });
}
```

This opens Salesforce's built-in document previewer which supports:
- PDF: full page rendering with zoom, navigation, and search
- DOCX/XLSX/PPTX: server-side converted preview (rendered by Salesforce)
- Images: full-size view with zoom
- Download button available in all preview modes

**Direct download button (stops event propagation to avoid triggering preview):**

```javascript
handleDocumentDownload(event) {
    event.stopPropagation();
    window.open(
        `/sfc/servlet.shepherd/version/download/${this.attachment.Content_Version_Id__c}`,
        '_blank'
    );
}
```

### 5E. Sticker Display

**Chat bubble with sticker image:**

```html
<template if:true={isSticker}>
    <div class="chat-sticker-container">
        <img
            src={stickerUrl}
            alt="Sticker"
            class="chat-sticker"
            loading="lazy"
        />
    </div>
</template>
```

**Size constraints:**
- Telegram stickers are 512x512 px maximum.
- Display at 200x200 px in chat bubble (CSS constrained).
- No click-to-preview needed (stickers are decorative, already small).

**URL:**

```javascript
get stickerUrl() {
    // Use thumbnail if available (animated sticker first frame), else direct download
    if (this.attachment.Thumbnail_CV_Id__c) {
        return `/sfc/servlet.shepherd/version/renditionDownload`
            + `?rendition=THUMB240BY180`
            + `&versionId=${this.attachment.Thumbnail_CV_Id__c}`;
    }
    return `/sfc/servlet.shepherd/version/download/${this.attachment.Content_Version_Id__c}`;
}
```

**Format handling by Go before upload:**
- Static stickers (WebP): Convert to PNG (WebP not reliably rendered by Salesforce thumbnail system).
- Animated stickers (TGS/Lottie): Render first frame to PNG. Upload PNG as ContentVersion. Animated playback is not supported in the LWC context.
- Video stickers (WebM): Extract first frame via ffmpeg. Upload frame as PNG.

**CSS:**

```css
.chat-sticker {
    max-width: 200px;
    max-height: 200px;
    object-fit: contain;
    border: none;
    background: transparent;
}
```

---

## 6. SOQL Query Design

### 6A. Loading Messages for a Chat Thread

**Primary query (existing fields + new Has_File__c, File_Count__c):**

```apex
@AuraEnabled(cacheable=true)
public static List<Messenger_Message__c> getChatMessages(Id chatId, Integer pageSize, Integer offset) {
    return [
        SELECT Id, Message_Text__c, Direction__c, Message_Type__c,
               Sender_Name__c, Sent_At__c, Delivery_Status__c,
               Is_Edited__c, Reply_To_External_ID__c,
               Has_File__c, File_Count__c, Contains_Inbound_Files__c,
               Media_MIME_Type__c
        FROM Messenger_Message__c
        WHERE Chat__c = :chatId
        WITH SECURITY_ENFORCED
        ORDER BY Sent_At__c DESC
        LIMIT :pageSize
        OFFSET :offset
    ];
}
```

This query does NOT join to ContentDocumentLink or Messenger_Attachment__c.
File data is loaded separately, only for messages that have files. This separation
is essential to avoid the N+1 problem and keep the primary chat query fast.

### 6B. Batch-Loading Files for Messages with Has_File__c = true

**Step 1: Collect message IDs that have files (in LWC JavaScript):**

```javascript
const messagesWithFiles = this.messages.filter(m => m.Has_File__c);
const messageIds = messagesWithFiles.map(m => m.Id);
```

**Step 2: Single batch query for all attachments (one SOQL for entire page):**

```apex
@AuraEnabled(cacheable=true)
public static Map<Id, List<AttachmentDTO>> getAttachmentsForMessages(List<Id> messageIds) {
    if (messageIds.isEmpty()) return new Map<Id, List<AttachmentDTO>>();

    List<Messenger_Attachment__c> attachments = [
        SELECT Id, Message__c, Content_Version_Id__c, Content_Document_Id__c,
               Thumbnail_CV_Id__c, Media_MIME_Type__c, File_Name__c,
               File_Size__c, Sort_Order__c, Duration_Seconds__c,
               Width__c, Height__c, Waveform_JSON__c
        FROM Messenger_Attachment__c
        WHERE Message__c IN :messageIds
        WITH SECURITY_ENFORCED
        ORDER BY Message__c, Sort_Order__c ASC
    ];

    // Group by message ID -> immutable DTO list per message
    Map<Id, List<AttachmentDTO>> result = new Map<Id, List<AttachmentDTO>>();
    for (Messenger_Attachment__c att : attachments) {
        if (!result.containsKey(att.Message__c)) {
            result.put(att.Message__c, new List<AttachmentDTO>());
        }
        result.get(att.Message__c).add(new AttachmentDTO(att));
    }
    return result;
}
```

**SOQL count for loading a chat page: exactly 2 queries total:**
1. `getChatMessages` -- load the page of messages
2. `getAttachmentsForMessages` -- load ALL attachments for ALL messages with files on that page

This avoids the N+1 problem entirely. Even with 50 messages and 30 having files, it is
still exactly 2 SOQL queries, well within the 100-query synchronous limit.

### 6C. Fallback: Direct ContentDocumentLink Query

For outbound messages where files are attached via `lightning-file-upload` BEFORE the
`Messenger_Attachment__c` record is created (race condition during send flow):

```apex
@AuraEnabled(cacheable=true)
public static List<FileDTO> getFilesForRecord(Id recordId) {
    List<ContentDocumentLink> links = [
        SELECT ContentDocumentId,
               ContentDocument.Title,
               ContentDocument.FileType,
               ContentDocument.ContentSize,
               ContentDocument.LatestPublishedVersionId
        FROM ContentDocumentLink
        WHERE LinkedEntityId = :recordId
        WITH SECURITY_ENFORCED
        ORDER BY ContentDocument.CreatedDate ASC
    ];

    List<FileDTO> files = new List<FileDTO>();
    for (ContentDocumentLink link : links) {
        files.add(new FileDTO(
            link.ContentDocument.LatestPublishedVersionId,
            link.ContentDocumentId,
            link.ContentDocument.Title,
            link.ContentDocument.FileType,
            link.ContentDocument.ContentSize
        ));
    }
    return files;
}
```

**Important:** ContentDocumentLink SOQL requires a filter on `ContentDocumentId` or
`LinkedEntityId` using `=` or `IN` operators. Unfiltered queries are not allowed.
This query filters by `LinkedEntityId` (indexed), which is the optimal pattern.

### 6D. @wire vs Imperative

| Query | Pattern | Rationale |
|---|---|---|
| `getChatMessages` | `@wire` with `cacheable=true` | Messages are read-heavy. Wire provides automatic caching and re-rendering. Invalidate on new message via `refreshApex()`. |
| `getAttachmentsForMessages` | **Imperative** | Called conditionally only when messages have files. Cannot use `@wire` because the input (`messageIds`) depends on the first query result. |
| `getFilesForRecord` | **Imperative** | Called on-demand for outbound draft preview or as fallback. |

**Wire adapter for messages with attachment loading:**

```javascript
@wire(getChatMessages, { chatId: '$chatId', pageSize: 50, offset: 0 })
wiredMessages(result) {
    this._wiredMessagesResult = result;
    if (result.data) {
        this.messages = result.data;
        this.loadAttachmentsForVisibleMessages();
    }
    if (result.error) {
        this.showToast('Error', 'Failed to load messages', 'error');
    }
}

async loadAttachmentsForVisibleMessages() {
    const idsWithFiles = this.messages
        .filter(m => m.Has_File__c)
        .map(m => m.Id);

    if (idsWithFiles.length === 0) {
        this.attachmentsByMessageId = {};
        return;
    }

    try {
        const attachmentMap = await getAttachmentsForMessages({ messageIds: idsWithFiles });
        // Create new object (immutability)
        this.attachmentsByMessageId = Object.freeze({ ...attachmentMap });
    } catch (error) {
        console.error('Failed to load attachments:', error);
    }
}
```

---

## 7. Components Added/Removed/Modified

### Go Middleware

| Component | Change | Details |
|---|---|---|
| `internal/media/r2.go` | **REMOVE** | R2 upload/download adapter. All S3-compatible API calls eliminated. |
| `internal/media/presigned.go` | **REMOVE** | SigV4 presigned URL generator. No longer needed. |
| `internal/media/hmac.go` | **REMOVE** | HMAC URL signing for inbound display. No longer needed. |
| `internal/salesforce/contentversion.go` | **ADD** | New adapter: `Upload()` (multipart POST), `Download()` (streaming GET VersionData), `QueryDocumentID()`. |
| `internal/salesforce/contentversion_test.go` | **ADD** | Unit tests with mock HTTP server. |
| `internal/media/thumbnail.go` | **ADD** | ffmpeg wrapper for video first-frame extraction. WebP-to-PNG conversion. TGS first-frame rendering. |
| `internal/media/thumbnail_test.go` | **ADD** | Tests with sample media files. |
| `internal/media/queue.go` | **MODIFY** | Queue destination changes from R2 object key to SF ContentVersion ID. Add API budget check via `/limits/` endpoint. |
| `internal/telegram/media.go` | **MODIFY** | Upload destination: R2 -> `contentversion.Upload()`. Download source: R2 -> `contentversion.Download()`. |
| `internal/config/config.go` | **MODIFY** | Remove R2 env vars (R2_ACCOUNT_ID, R2_ACCESS_KEY, R2_SECRET_KEY, R2_BUCKET). Add SF_MEDIA_RATE_LIMIT, SF_API_BUDGET_THRESHOLD. |
| `migrations/xxx_drop_media_files.sql` | **ADD** | Drop `media_files` table from PostgreSQL. |
| `go.mod` | **MODIFY** | Remove AWS SDK S3 dependencies. Run `go mod tidy`. |

### Salesforce (Apex)

| Component | Change | Details |
|---|---|---|
| `MessengerController.cls` | **MODIFY** | Add `getAttachmentsForMessages()`, `getFilesForRecord()` methods. Remove R2 CDN URL construction. Add CRUD/FLS checks on ContentVersion queries. |
| `MessengerOutboundService.cls` | **MODIFY** | Add ContentVersion query before callout to Go. Pass CV IDs instead of R2 URLs. |
| `ContentDocumentLinkTrigger.trigger` | **ADD** | New trigger on ContentDocumentLink (after insert, after delete). Filters for Messenger_Message__c links by key prefix. |
| `ContentDocumentLinkTriggerHandler.cls` | **ADD** | Handler: aggregate count per message, update `Has_File__c` and `File_Count__c`. Bulk-safe (up to 200 records). Uses `without sharing` to ensure count updates regardless of user context. |
| `ContentDocumentLinkTriggerHandlerTest.cls` | **ADD** | Test class: insert CV with FirstPublishLocationId, verify Has_File__c. Delete scenario. Bulk 200-record scenario. |
| `OutboundMediaQueueable.cls` | **ADD** | Queueable Apex for async callout to Go with ContentVersion IDs. Retry via chaining on transient failure. |
| `OutboundMediaQueueableTest.cls` | **ADD** | Test with HttpCalloutMock. |
| `AttachmentDTO.cls` | **ADD** | Lightweight DTO wrapping Messenger_Attachment__c fields for LWC. |
| `FileDTO.cls` | **ADD** | DTO for ContentDocumentLink-based file metadata. |

### Salesforce (LWC)

| Component | Change | Details |
|---|---|---|
| `messengerChat` | **MODIFY** | Add `loadAttachmentsForVisibleMessages()`. Replace CDN URL rendering with `/sfc/servlet.shepherd/` URLs. Add album grid layout CSS. Add `Has_File__c` conditional rendering. |
| `messengerMediaViewer` | **REWRITE** | Was R2/CDN lightbox. Rewrite to use NavigationMixin filePreview + custom video modal with `<video>` element. |
| `messengerFileUpload` | **ADD** | Wraps `<lightning-file-upload>` with file type validation, accepted formats, progress display, and pending file list. |
| `messengerVoicePlayer` | **ADD** | Audio player with waveform visualization using SVG/CSS bars from `Waveform_JSON__c`. |
| `messengerDocumentCard` | **ADD** | Document display card: SLDS doctype icon, filename, size, download button, click-to-preview. |
| `messengerStickerDisplay` | **ADD** | Sticker display with 200x200 CSS constraints. Handles PNG (converted from WebP/TGS by Go). |
| `messengerStorageMonitor` | **ADD** | Admin-only component showing file storage usage via `System.OrgLimits`. Alert thresholds at 70%, 85%, 95%. |
| `centrifugoClient` | **NO CHANGE** | WebSocket client unchanged. |

### Infrastructure (Removed)

| Component | Change | Details |
|---|---|---|
| Cloudflare R2 bucket | **REMOVE** | No longer needed. All media in Salesforce ContentVersion. |
| Cloudflare Worker (CDN proxy) | **REMOVE** | No longer needed. Same-origin serving via servlet.shepherd. |
| CSP Trusted Site (R2 domain) | **REMOVE** | No external media domain to trust. |
| CSP Trusted Site (Worker CDN) | **REMOVE** | No external CDN domain. |
| CORS config (R2 bucket) | **REMOVE** | No cross-origin uploads. |
| CORS config (SF for R2) | **REMOVE** | No bilateral CORS needed. |
| PostgreSQL `media_files` table | **REMOVE** | R2 object keys no longer tracked in PostgreSQL. |
| Media orphan cleanup job | **REMOVE** | No orphaned R2 objects to clean up. |

### Configuration

| Component | Change | Details |
|---|---|---|
| CSP Trusted Site (Centrifugo) | **KEEP** | Still needed for WebSocket connection. |
| Connected App (OAuth) | **KEEP** | Still needed for Go -> SF auth. |
| Permission Set (Messenger_Agent) | **MODIFY** | Add: ContentVersion (Read, Create), ContentDocumentLink (Read, Create). Add new fields to FLS. |
| Permission Set (Messenger_Admin) | **MODIFY** | Add: ContentVersion (Read, Create, Delete), ContentDocumentLink (Read, Create, Delete), ContentDocument (Read, Delete). Add new fields to FLS. |
| File Upload and Download Security | **DOCUMENT** | Post-install step: set .mp4, .webm, .mov to "Execute in Browser" for inline video playback. |

---

## 8. ContentDocumentLink Trigger Design

### Trigger Definition

```apex
// force-app/main/default/triggers/ContentDocumentLinkTrigger.trigger
trigger ContentDocumentLinkTrigger on ContentDocumentLink (after insert, after delete) {
    ContentDocumentLinkTriggerHandler.handleFileLinks(
        Trigger.isInsert ? Trigger.new : null,
        Trigger.isDelete ? Trigger.old : null
    );
}
```

### Handler Implementation

```apex
// force-app/main/default/classes/ContentDocumentLinkTriggerHandler.cls
public without sharing class ContentDocumentLinkTriggerHandler {

    // Cache the key prefix for Messenger_Message__c (3-character prefix, e.g., "a0B")
    private static final String MESSAGE_KEY_PREFIX =
        tgint__Messenger_Message__c.SObjectType.getDescribe().getKeyPrefix();

    /**
     * Handles ContentDocumentLink insert/delete events.
     * Filters for links to Messenger_Message__c records only.
     * Updates Has_File__c and File_Count__c on the parent message.
     *
     * Bulk-safe: handles up to 200 records per trigger invocation.
     * Uses "without sharing" to ensure file count updates succeed
     * regardless of the uploading user's sharing access to the message.
     */
    public static void handleFileLinks(
        List<ContentDocumentLink> insertedLinks,
        List<ContentDocumentLink> deletedLinks
    ) {
        // 1. Collect affected Messenger_Message__c IDs
        Set<Id> affectedMessageIds = new Set<Id>();
        List<ContentDocumentLink> relevantLinks =
            insertedLinks != null ? insertedLinks : deletedLinks;

        for (ContentDocumentLink cdl : relevantLinks) {
            // LinkedEntityId is polymorphic. Filter by key prefix to identify
            // Messenger_Message__c records without a describe call per record.
            String linkedIdStr = String.valueOf(cdl.LinkedEntityId);
            if (linkedIdStr != null && linkedIdStr.startsWith(MESSAGE_KEY_PREFIX)) {
                affectedMessageIds.add(cdl.LinkedEntityId);
            }
        }

        if (affectedMessageIds.isEmpty()) {
            return; // No Messenger_Message__c links in this batch. Exit early.
        }

        // 2. Count current ContentDocumentLinks per message.
        //    Single aggregate query handles the entire batch.
        //    Governor limit impact: 1 SOQL query.
        Map<Id, Integer> fileCountByMessage = new Map<Id, Integer>();

        for (AggregateResult ar : [
            SELECT LinkedEntityId, COUNT(Id) cnt
            FROM ContentDocumentLink
            WHERE LinkedEntityId IN :affectedMessageIds
            GROUP BY LinkedEntityId
        ]) {
            fileCountByMessage.put(
                (Id) ar.get('LinkedEntityId'),
                (Integer) ar.get('cnt')
            );
        }

        // 3. Build update list.
        //    For messages not in the aggregate result (all links deleted),
        //    set count to 0 and Has_File__c to false.
        List<tgint__Messenger_Message__c> messagesToUpdate =
            new List<tgint__Messenger_Message__c>();

        for (Id messageId : affectedMessageIds) {
            Integer count = fileCountByMessage.containsKey(messageId)
                ? fileCountByMessage.get(messageId)
                : 0;

            messagesToUpdate.add(new tgint__Messenger_Message__c(
                Id = messageId,
                tgint__Has_File__c = count > 0,
                tgint__File_Count__c = count
            ));
        }

        // 4. Single bulk DML.
        //    Governor limit impact: 1 DML statement.
        if (!messagesToUpdate.isEmpty()) {
            update messagesToUpdate;
        }
    }
}
```

### Governor Limit Analysis

| Limit | Consumed | Budget (Sync) | Percentage |
|---|---|---|---|
| SOQL queries | 1 (aggregate) | 100 | 1% |
| DML statements | 1 (bulk update) | 150 | 0.67% |
| Records retrieved | Up to 200 (matching batch size) | 50,000 | 0.4% |
| Heap size | Minimal (IDs + integers only, no blob data) | 6 MB | Negligible |
| CPU time | Sub-millisecond per record | 10,000 ms | Negligible |

**Bulk safety:** The trigger processes up to 200 `ContentDocumentLink` records in a single
invocation (Salesforce's standard trigger batch size). The aggregate SOQL and bulk DML
pattern ensures the governor limits scale linearly, not quadratically.

### Edge Cases

| Scenario | Behavior |
|---|---|
| File linked to non-Message record (Account, Contact, etc.) | Filtered out by key prefix check. No action taken. Zero performance impact on non-MessageForge file operations. |
| Multiple files uploaded simultaneously (album) | Each ContentDocumentLink fires the trigger. Salesforce may batch them, so the trigger processes all at once. Final `File_Count__c` is correct because it re-counts via aggregate query (not increment). |
| File deleted from Message | `after delete` context. Aggregate query returns reduced count. `Has_File__c` set to `false` if count reaches 0. |
| File linked to both Message and Chat records | Only the Message link triggers the update. Chat links are ignored (different key prefix). |
| Other packages with CDL triggers | Each namespace gets its own trigger. No conflict. Execution order is non-deterministic but both triggers run independently. |
| ContentDocumentLink does not support addError() | Known Salesforce limitation: custom error messages via `addError()` do not work in ContentDocumentLink triggers. If validation is needed, implement it in a before-insert trigger on the parent record instead. |

### Managed Package Considerations

1. **Standard object trigger:** Managed packages can include triggers on standard objects like
   `ContentDocumentLink`. The trigger is namespaced (`tgint__ContentDocumentLinkTrigger`).

2. **One trigger per namespace per object:** Salesforce allows one trigger per namespace per
   object. If the customer has their own trigger on `ContentDocumentLink`, it runs in a
   separate context (no conflict).

3. **Sharing context:** The handler uses `without sharing` to ensure file count updates succeed
   regardless of the uploading user's access level to the message record. This is safe because
   the trigger only updates two metadata fields (`Has_File__c`, `File_Count__c`) -- it never
   exposes or modifies message content.

4. **CRUD/FLS in triggers:** Triggers run in system context, so CRUD/FLS enforcement is not
   required. However, for AppExchange security review, document this bypass with justification:
   "System operation -- trigger updates denormalized count fields that must reflect ground truth
   regardless of the triggering user's permissions."

5. **Test coverage:** The trigger must have 75%+ test coverage for managed package deployment.
   Test by inserting a `ContentVersion` with `FirstPublishLocationId` pointing to a test
   `Messenger_Message__c` record, then asserting `Has_File__c = true`.

6. **ContentDocumentLink bulkification known issue:** Salesforce has a known issue where
   ContentDocumentLink doesn't fully support bulkification. The aggregate query pattern
   used here is resilient to this because it re-counts from the database rather than relying
   on trigger context size assumptions.

---

## Appendix A: URL Reference

| Purpose | URL Pattern | Notes |
|---|---|---|
| Small thumbnail (120x90) | `/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB120BY90&versionId={cvId}` | Chat list preview, very small |
| Medium thumbnail (240x180) | `/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB240BY180&versionId={cvId}` | Grid view, sticker display |
| Large thumbnail (720x480) | `/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB720BY480&versionId={cvId}` | Chat bubble display (recommended default) |
| Full file download | `/sfc/servlet.shepherd/version/download/{cvId}` | Video/audio source, document download, direct binary |
| Document download by doc ID | `/sfc/servlet.shepherd/document/download/{cdId}` | Alternative download by ContentDocument ID |
| Multiple file download (zip) | `/sfc/servlet.shepherd/version/download/{cvId1}/{cvId2}/...` | Bulk download as ZIP, CVIds separated by `/` |
| NavigationMixin filePreview | `{ type: 'standard__namedPage', attributes: { pageName: 'filePreview' }, state: { recordIds: 'cdId1,cdId2', selectedRecordId: 'cdId' } }` | Opens native Salesforce preview modal with navigation |

## Appendix B: ContentDocumentLink ShareType/Visibility Reference

| Field | Value | Meaning | Our Usage |
|---|---|---|---|
| `ShareType` | `V` | Viewer -- can view but not edit | Default for inbound media |
| `ShareType` | `C` | Collaborator -- can edit | Not used |
| `ShareType` | `I` | Inferred -- inherits from parent record sharing | **Recommended for ISV.** Access follows `Messenger_Message__c` sharing rules. |
| `Visibility` | `AllUsers` | Visible to all users with record access | **Recommended for internal orgs.** |
| `Visibility` | `InternalUsers` | Visible only to internal users (excludes portal/community) | Use if customer has Experience Cloud portals and wants to restrict media visibility. |
| `Visibility` | `SharedUsers` | Visible to users who can see the feed | Typically for Chatter; not applicable here. |

**ISV recommendation:** When Go uploads ContentVersion with `FirstPublishLocationId`, the
auto-created ContentDocumentLink uses `ShareType = 'V'` and the org's default Visibility
setting. For maximum compatibility, accept the defaults. If specific control is needed,
create the ContentDocumentLink manually with `ShareType = 'I'` to inherit sharing from
the linked `Messenger_Message__c` record.

## Appendix C: AppExchange Security Review Checklist for ContentVersion

| Requirement | Implementation | Status |
|---|---|---|
| CRUD check before ContentVersion insert | `Schema.sObjectType.ContentVersion.isCreateable()` in Apex; Go uses OAuth token with profile permissions | Required |
| FLS check on ContentVersion queries | `WITH SECURITY_ENFORCED` on all SOQL queries in `@AuraEnabled` methods | Required |
| FLS check on query results | `Security.stripInaccessible(AccessType.READABLE, records)` for query results returned to LWC | Required |
| Permission Set grants | Messenger_Agent: Read+Create on ContentVersion, Read+Create on ContentDocumentLink | Required |
| File type validation | Go validates MIME type against allowlist before upload. Block executables (.exe, .bat, .sh, .js) | Recommended |
| Bypass documentation | Trigger handler runs `without sharing` for count updates -- document in security review submission | Required |
| Salesforce Code Analyzer | Run with `--rule-selector AppExchange --rule-selector Recommended:Security` against all Apex | Required |

## Appendix D: Storage and API Budget Reference

### File Storage Per Edition

| Edition | Base (org-wide) | Per User License | Example: 100 users |
|---|---|---|---|
| Professional | 10 GB | 612 MB | ~70 GB |
| Enterprise | 10 GB | 2 GB | **210 GB** |
| Unlimited | 10 GB | 2 GB | **210 GB** |

Additional storage: **$5/GB/month** (~$60/GB/year).

Salesforce-generated thumbnails/renditions do NOT count against storage.
Each ContentVersion's VersionData blob counts. Multiple versions of the same file count separately.

### API Call Budget Per Edition

| Edition | Base | Per User | Example: 50 users |
|---|---|---|---|
| Enterprise | 100,000/day | 1,000/user/day | **150,000/day** |
| Unlimited | 100,000/day | 5,000/user/day | **350,000/day** |

Additional API calls: **$25/month per 10,000/day**.

### Media Upload Modeling

| Volume | API calls/day | % of Enterprise (50u) | % of Unlimited (50u) |
|---|---|---|---|
| 50 media/day (light) | ~200 | 0.13% | 0.06% |
| 500 media/day (moderate) | ~2,000 | 1.3% | 0.6% |
| 5,000 media/day (heavy) | ~20,000 | 13.3% | 5.7% |
| 50,000 media/day (extreme) | ~200,000 | 133% (OVER) | 57% |

Note: Each media message requires ~4 API calls (message create + file upload + doc ID query +
attachment create). Downloads for outbound add 1 API call each.

### What Happens When Storage Is Exceeded

1. Grace period up to ~110% of allocated storage.
2. Beyond 110%, uploads are blocked: `STORAGE_LIMIT_EXCEEDED` error.
3. Automated processes (triggers, API inserts) that create files will fail.
4. Salesforce sends admin notifications when approaching limits.
5. Resolution: purchase additional storage or archive old files.

---

## Appendix E: Unresolved Questions (Require Scratch Org Testing)

1. **HTTP Range request support:** Does `/sfc/servlet.shepherd/version/download/` return `Accept-Ranges: bytes` header? If yes, HTML5 video seeking works natively. SalesforceLabs VideoViewer suggests it does, but no official documentation confirms this.

2. **ContentDocumentLink trigger firing on REST API uploads:** Verify that when Go middleware uploads a ContentVersion with FirstPublishLocationId, the auto-created ContentDocumentLink fires the `after insert` trigger correctly.

3. **File Upload and Download Security defaults:** Check whether `.mp4`, `.webm`, `.mov` are set to "Execute in Browser" or "Download" by default in a fresh scratch org.

4. **Concurrent upload throughput:** Benchmark how many parallel ContentVersion REST API uploads the Salesforce org can sustain before hitting rate limits.

5. **Lightning file preview for video:** Test whether the NavigationMixin `filePreview` page correctly handles video files with a playback UI, or only shows a download button.

---

## Web Research Sources

- [ContentVersion Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_contentversion.htm)
- [ContentDocumentLink Object Reference](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_contentdocumentlink.htm)
- [Insert or Update Blob Data (REST API)](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_sobject_insert_update_blob.htm)
- [lightning-file-upload Component](https://developer.salesforce.com/docs/component-library/bundle/lightning-file-upload)
- [Open Files (NavigationMixin filePreview)](https://developer.salesforce.com/docs/platform/lwc/guide/use-open-files.html)
- [SalesforceLabs VideoViewer](https://github.com/SalesforceLabs/VideoViewer)
- [File Preview in LWC](https://www.salesforcecodecrack.com/2019/09/file-preview-in-lwc.html)
- [URL Hacking: File Preview & Downloads](https://medium.com/@vishwakarmaas27/the-art-of-url-hacking-in-salesforce-part-iv-file-preview-downloads-3cab6db87dd0)
- [Default Visibility for Files Shared on Records](https://help.salesforce.com/s/articleView?id=000384243&language=en_US&type=1)
- [ISVforce Guide](https://developer.salesforce.com/docs/atlas.en-us.packagingGuide.meta/packagingGuide/about_managed_packages.htm)
