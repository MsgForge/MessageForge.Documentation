# Implementation Plan: MessageForge Phase 1 LWC Managed Package

## Affected files

| File | Action | Reason |
| ---- | ------ | ------ |
| `MessageForge.SalesForce/force-app/main/default/objects/` (21+ dirs) | Create | Custom objects, fields, relationships |
| `MessageForge.SalesForce/force-app/main/default/customMetadata/` | Create | Channel_Type__mdt, Encryption_Key__mdt seed records |
| `MessageForge.SalesForce/force-app/main/default/platformEvents/` | Create | 3 platform events |
| `MessageForge.SalesForce/force-app/main/default/permissionsets/` | Create | Admin, Agent, Viewer permission sets |
| `MessageForge.SalesForce/force-app/main/default/lwc/` (12+ dirs) | Create | All LWC components |
| `MessageForge.SalesForce/force-app/main/default/classes/` (20+ files) | Create | Controllers, services, triggers, test classes |
| `MessageForge.SalesForce/force-app/main/default/triggers/` (5 files) | Create | All Apex triggers |
| `MessageForge.SalesForce/force-app/main/default/namedCredentials/` | Create | Per-provider credential metadata |

---

## 1. Design Decisions

### Decision 1: Cross-channel timeline

**Decision: Separate tabs per channel (prototype's current design). No unified timeline in Phase 1.**

Rationale: A unified timeline requires a new `Conversation__c` parent object, a trigger to auto-group chats by Lead/Contact, and re-parenting logic for orphan chats. This adds schema complexity and creates ambiguity (what happens when the same person messages from two WhatsApp numbers?). The tab-per-channel model in the prototype already works, requires zero schema changes, and gives agents clear context about which channel they are replying on. The embedded `messengerChat` in `mode='embedded'` uses `lightning-tabset` with one tab per `Messenger_Chat__c` linked to the record. Phase 2 can introduce `Conversation__c` with timeline merging once the base is stable.

**Schema impact: None.**

### Decision 2: Lead/Contact matching

**Decision: Phone-based auto-link with manual override UI. First match wins; prefer Contact over Lead.**

Algorithm on inbound new chat creation:
1. Normalize sender identifier to E.164 for phone-based channels (WhatsApp, SMS, Viber). For Telegram/Messenger/Instagram, use the platform user ID.
2. SOQL `Contact` by `Phone`, `MobilePhone`, or `HomePhone` matching normalized value. If found, link `Contact__c`.
3. If no Contact match, SOQL `Lead` by `Phone` or `MobilePhone`. If found, link `Lead__c`.
4. If no match, create chat with both lookups null (orphan).

Conflict-resolution UI: The `messengerChat` header displays a "Link" button for orphan chats. Clicking it opens a `lightning-record-picker` that searches Lead and Contact. For already-linked chats, a small dropdown in the header allows "Re-link to different record" or "Unlink". This is a lightweight inline action, not a modal.

**Schema impact: None -- existing `Lead__c` and `Contact__c` lookups on `Messenger_Chat__c` are sufficient.**

### Decision 3: High-volume mode

**Decision: Platform Events only in Phase 1. No polling fallback. Document the 1,000 msgs/day soft ceiling.**

Rationale: Adding a polling mode doubles the controller API surface, introduces consistency bugs (stale data vs. real-time), and complicates every LWC component. The 10K PE daily limit on Enterprise edition is a known constraint. For higher-volume customers, the mitigation is: (a) recommend Unlimited edition (50K), (b) sell PE add-on allocations, (c) scope the PE subscription narrowly (only subscribe when the user is actively viewing a chat, unsubscribe on component disconnect). The LWC will implement lazy subscription -- subscribe to PE channels only in `connectedCallback()` and unsubscribe in `disconnectedCallback()`.

Phase 2 can add SOQL-based polling as a toggle using `MessengerController.getRecentMessages()` which we define anyway for initial data load. But Phase 1 does not implement the polling timer.

**Schema impact: None.**

### Decision 4: Chat list scaling

**Decision: Server-side cursor pagination (20 chats per page) + search + channel/status filters. No virtual scrolling in Phase 1.**

The `MessengerController.getChatList()` method accepts filters (channel IDs, status, search text) and pagination params (pageSize, lastMessageAt cursor). Returns the 20 most recently active chats matching the filter. A "Load more" button at the bottom of the chat list loads the next page. Search is server-side SOQL `LIKE` on `Chat_Title__c` and `Last_Message_Preview__c`.

This avoids the complexity of virtual scrolling (which requires exact row-height calculations and a scroll container library) while keeping SOQL queries efficient with indexed `Last_Message_At__c` ORDER BY.

**Schema impact: None -- `Last_Message_At__c` already exists on `Messenger_Chat__c`.**

### Decision 5: Optimistic outbound rendering

**Decision: Confirmed. Insert pending, return ID, LWC shows immediately, PE updates status.**

Flow:
1. LWC calls `MessengerController.sendMessage(chatId, text, attachmentIds)`.
2. Apex inserts `Messenger_Message__c` with `Direction__c='outbound'`, `Delivery_Status__c='pending'`, returns the record ID.
3. Apex enqueues `MessengerOutboundQueueable`.
4. LWC immediately appends the message to the UI with a "sending..." indicator (clock icon).
5. When `tgint__Message_Delivery_Status__e` arrives with the matching `Message_SF_ID__c`, LWC updates the status icon: pending -> sent (single check) -> delivered (double check) -> read (double check blue) or failed (red X).

The LWC also handles the edge case where the PE arrives before the `sendMessage()` promise resolves -- it buffers PEs for unrecognized message IDs and replays them after the imperative call returns.

**Schema impact: Add `Message_SF_ID__c` (Text 18) to `tgint__Message_Delivery_Status__e` so the LWC can correlate PE to the optimistically-rendered message. The existing `Message_External_ID__c` is the provider's ID which is not yet known at send time.**

### Decision 6: Template picker UX

**Decision: Add `Variable_Schema_JSON__c` to `Messenger_Template__c`. Template picker is a child component of the composer.**

The `messengerTemplatePicker` component:
1. Shows when the agent clicks a "Templates" button in the composer (visible only for WhatsApp channels and when outside the 24h session window).
2. Lists approved templates from `Messenger_Template__c` filtered by `Channel__c` and `Approval_Status__c = 'approved'` and `Is_Active__c = true`.
3. Selecting a template shows the body with `{{1}}`, `{{2}}` placeholders highlighted.
4. For each placeholder, renders a `lightning-input` for variable substitution.
5. Real-time preview updates as variables are filled.
6. "Send Template" button calls `MessengerController.sendTemplate(chatId, templateId, variableMap)`.

The `Variable_Schema_JSON__c` field stores a JSON array describing each variable: `[{"index": 1, "label": "Customer Name", "example": "John"}, ...]`. This is populated during template sync from Meta's Graph API.

**Schema impact: New field `Variable_Schema_JSON__c` (Long Text Area, 5000) on `Messenger_Template__c`.**

### Decision 7: Wizard re-entry

**Decision: Resume from last completed step. Draft rows are retained and cleaned up after 30 days.**

When a channel has `Status__c = 'draft'`:
- The channel list shows a "Resume Setup" action button.
- Clicking it opens the wizard at the step after the last completed one. The wizard determines the last completed step by checking which child records exist:
  - App record exists (`*_App__c`) -> step 2 is done, resume at step 3.
  - Config record exists (`*_Config__c`) -> step 3 is done, resume at step 4.
  - No child records -> resume at step 1.
- Each wizard step is idempotent: if the user re-runs step 2, it upserts the App record rather than creating a duplicate.
- A scheduled batch job (Phase 2) will clean up draft channels older than 30 days. Phase 1 relies on manual cleanup.

**Schema impact: None -- `Status__c = 'draft'` and child record checks are sufficient.**

### Decision 8: New message notifications

**Decision: Custom Notification via `CustomNotificationType` for the assigned user. Standard bell notification.**

When a new inbound message is inserted and `Messenger_Chat__c.Assigned_User__c` is populated:
1. A trigger on `Messenger_Message__c` (after insert, for `Direction__c = 'inbound'`) checks if the assigned user currently has the LWC open (we cannot check this, so we always send).
2. It calls `Messaging.CustomNotification` with title "New message from {Sender_Name__c}" and body = first 100 chars of the message text.
3. The notification type is `Messenger_New_Message` (deployed in the package).
4. Clicking the notification navigates to the `Messenger_Chat__c` record page (which hosts the embedded chat).

This is lightweight, uses native Salesforce bell notifications, requires no email infrastructure, and works on both desktop and mobile.

**Schema impact: New `CustomNotificationType` metadata: `Messenger_New_Message`.**

### Decision 9: Mobile

**Decision: Explicitly OUT of Phase 1 scope.**

The LWC components will use responsive SLDS utilities where trivial (e.g., `slds-size_1-of-1` on small screens), but no dedicated mobile layout work. The `messengerChat` component will not be tested or optimized for the Salesforce Mobile App. The chat pane on mobile would require a completely different navigation pattern (stack-based instead of side-by-side). Phase 2 will address mobile with a dedicated `messengerChatMobile` component or responsive breakpoints.

**Schema impact: None.**

### Decision 10: Media thumbnails

**Decision: Lazy-load full originals inline. No thumbnail generation in Phase 1.**

Rationale: Generating thumbnails in Apex requires either (a) a Queueable with an image processing library (none native to Apex), (b) a canvas-based resize in LWC (browser-only, cannot persist), or (c) an external service (out of scope). The simplest approach is to load the full image via `ContentVersion` download URL and let the browser handle display scaling via CSS `max-width`/`max-height` on the `<img>` tag.

For performance, the `messengerChatBubble` component uses `loading="lazy"` on `<img>` tags so images below the fold do not load until scrolled into view. This is the browser-native lazy loading API, zero code overhead.

The `Messenger_Attachment__c.Thumbnail_URL__c` field remains in the schema but is null in Phase 1. Phase 2 can populate it with an external thumbnail service.

**Schema impact: None -- `Thumbnail_URL__c` already exists and stays null.**

---

## 2. LWC Component Tree

### `channelSetupWizard`

| Property | Value |
|----------|-------|
| **@api** | `channelId` (String, optional -- for resume/edit mode) |
| **@wire** | None |
| **Custom events fired** | `channelsaved` (detail: `{ channelId: String, status: String }`) |
| **Custom events handled** | None |
| **Apex methods** | `ChannelSetupController.listChannelTypes`, `ChannelSetupController.getExistingChannel`, `ChannelSetupController.createApp`, `ChannelSetupController.createChannel`, `ChannelSetupController.testConnection`, `ChannelSetupController.activateChannel`, `ChannelSetupController.registerWebhook` |
| **PE subscriptions** | None |
| **Children** | `channelTypeSelector` (step 1), `channelAppForm` (step 2), `channelConfigForm` (step 3), `channelConnectionTest` (step 4), `channelActivationForm` (step 5) |

### `channelTypeSelector`

| Property | Value |
|----------|-------|
| **@api** | `selectedType` (String), `channelTypes` (Array) |
| **Custom events fired** | `typeselect` (detail: `{ type: String }`) |
| **Children** | None (renders card grid internally) |

### `channelAppForm`

| Property | Value |
|----------|-------|
| **@api** | `channelType` (String), `appData` (Object, optional for edit) |
| **Custom events fired** | `appsubmit` (detail: `{ appName, apiId, apiHash, ownerEmail }`) |
| **Children** | Uses `lightning-input`, `lightning-combobox` |

### `channelConfigForm`

| Property | Value |
|----------|-------|
| **@api** | `channelType` (String), `configData` (Object, optional for edit), `appId` (String) |
| **Custom events fired** | `configsubmit` (detail: `{ channelName, handle, token, assigneeId, autoLinkStrategy }`) |
| **Children** | Uses `lightning-input`, `lightning-combobox`, `lightning-record-picker` |

### `channelConnectionTest`

| Property | Value |
|----------|-------|
| **@api** | `channelId` (String) |
| **Custom events fired** | `testcomplete` (detail: `{ success: Boolean, details: String }`) |
| **Apex methods** | `ChannelSetupController.testConnection` |
| **Children** | None |

### `channelActivationForm`

| Property | Value |
|----------|-------|
| **@api** | `channelId` (String), `webhookUrl` (String) |
| **Custom events fired** | `activate` (detail: `{ channelId: String }`) |
| **Children** | Uses `lightning-input` (read-only), `lightning-combobox` |

### `messengerChannelList`

| Property | Value |
|----------|-------|
| **@api** | None |
| **@wire** | None (imperative calls for refresh) |
| **Custom events fired** | `openchannel` (detail: `{ channelId }`), `editchannel` (detail: `{ channelId }`), `setupchannel` (detail: `{ channelId }`) |
| **Custom events handled** | `channelsaved` (from wizard) |
| **Apex methods** | `ChannelSetupController.listChannels`, `ChannelSetupController.toggleActive`, `ChannelSetupController.softDeleteChannel`, `ChannelSetupController.hardDeleteChannel` |
| **PE subscriptions** | None |
| **Children** | `channelSetupWizard`, `channelAuditLog` |

### `channelAuditLog`

| Property | Value |
|----------|-------|
| **@api** | `channelId` (String, optional -- null for all channels) |
| **Apex methods** | `ChannelSetupController.getRecentAuditLogs` |
| **Children** | None (renders `lightning-datatable`) |

### `messengerChat`

| Property | Value |
|----------|-------|
| **@api** | `recordId` (String, optional -- Lead/Contact ID for embedded mode), `mode` (String: 'full' or 'embedded', default 'full') |
| **@wire** | None |
| **Custom events fired** | None (top-level container) |
| **Custom events handled** | `chatselect` (from chat list), `messagesent` (from composer), `mediaclick` (from bubble), `templateselect` (from template picker), `linkrecord` (from header) |
| **Apex methods** | `MessengerController.getChatList`, `MessengerController.getRecentMessages`, `MessengerController.sendMessage`, `MessengerController.markAsRead`, `MessengerController.linkToRecord`, `MessengerController.getChatsForRecord` |
| **PE subscriptions** | `/event/tgint__Inbound_Message__e`, `/event/tgint__Message_Delivery_Status__e` |
| **Children** | `messengerChannelFilter`, `messengerChatList` (internal list), `messengerChatThread`, `messengerComposer`, `messengerMediaViewer` |

### `messengerChannelFilter`

| Property | Value |
|----------|-------|
| **@api** | `channels` (Array of channel objects), `selectedType` (String), `selectedChannelId` (String) |
| **Custom events fired** | `filterchange` (detail: `{ channelType: String, channelId: String }`) |
| **Children** | None (renders chip buttons + `lightning-combobox`) |

### `messengerChatList`

| Property | Value |
|----------|-------|
| **@api** | `chats` (Array), `activeChatId` (String) |
| **Custom events fired** | `chatselect` (detail: `{ chatId: String }`) |
| **Children** | None (renders list items internally) |

### `messengerChatThread`

| Property | Value |
|----------|-------|
| **@api** | `messages` (Array), `chatId` (String) |
| **Custom events fired** | `mediaclick` (detail: `{ attachmentId, contentVersionId, mimeType }`) |
| **Custom events handled** | None |
| **Children** | `messengerChatBubble` (for each message) |

### `messengerChatBubble`

| Property | Value |
|----------|-------|
| **@api** | `message` (Object: {id, direction, type, text, time, deliveryStatus, attachments, isEdited, lastError, replyToText, senderName}) |
| **Custom events fired** | `mediaclick` (detail: `{ attachmentId, contentVersionId, mimeType }`) |
| **Children** | None |
| **Notes** | Renders text, image, video, audio, document, sticker, template, album types. Renders `media_too_large_phase2` placeholder. Shows delivery status icons. Shows edit indicator. Shows reply-to quote block. |

### `messengerComposer`

| Property | Value |
|----------|-------|
| **@api** | `chatId` (String), `channelType` (String), `channelId` (String), `disabled` (Boolean) |
| **Custom events fired** | `messagesent` (detail: `{ text: String, attachmentIds: Array }`), `templaterequest` (detail: `{}`) |
| **Children** | `messengerTemplatePicker` (conditional, for WhatsApp) |
| **Notes** | Enter to send, Shift+Enter for newline. File upload via `lightning-file-upload`. Template button visible only for WhatsApp channels. |

### `messengerTemplatePicker`

| Property | Value |
|----------|-------|
| **@api** | `channelId` (String) |
| **Custom events fired** | `templateselect` (detail: `{ templateId: String, variables: Map<Integer, String> }`), `templatecancel` (detail: `{}`) |
| **Apex methods** | `MessengerController.getTemplates` |
| **Children** | None (renders template list + variable input form) |

### `messengerMediaViewer`

| Property | Value |
|----------|-------|
| **@api** | `contentVersionId` (String), `mimeType` (String), `fileName` (String) |
| **Custom events fired** | `close` (detail: `{}`) |
| **Children** | None (lightbox overlay with `<img>` / `<video>` / `<audio>` / download link) |

### `messengerInbox`

| Property | Value |
|----------|-------|
| **@api** | None (utility bar component) |
| **Custom events fired** | `openchat` (via `NavigationMixin` -- navigates to record) |
| **Apex methods** | `MessengerController.getInboxChats`, `MessengerController.sendQuickReply` |
| **PE subscriptions** | `/event/tgint__Inbound_Message__e` (for badge update) |
| **Children** | None (renders tab content internally) |
| **Notes** | Three tabs: New (unread > 0), Live (last message outbound), Pending (last message inbound). Quick reply input at bottom. |

### `messengerRecordLinker`

| Property | Value |
|----------|-------|
| **@api** | `chatId` (String), `currentLeadId` (String), `currentContactId` (String) |
| **Custom events fired** | `linkrecord` (detail: `{ chatId, recordId, recordType }`) |
| **Apex methods** | `MessengerController.linkToRecord` |
| **Children** | Uses `lightning-record-picker` |

---

## 3. Apex Controller API

### `ChannelSetupController.cls`

#### `listChannelTypes()`
- **Parameters:** None
- **Return type:** `List<ChannelTypeWrapper>` where `ChannelTypeWrapper` = `{ apiName: String, label: String, messenger: String, protocol: String, configObjectName: String, supportsOutbound: Boolean, isPhase1: Boolean }`
- **SOQL:** `SELECT DeveloperName, MasterLabel, Messenger__c, Protocol__c, Config_Object_Name__c, Supports_Outbound__c FROM tgint__Channel_Type__mdt`
- **DML:** None
- **Governor impact:** 1 SOQL, 0 DML

#### `listChannels(String searchText, String channelType, String status)`
- **Parameters:** `searchText` (String, nullable), `channelType` (String, nullable), `status` (String, nullable)
- **Return type:** `List<ChannelListWrapper>` = `{ id, name, subLabel, channelType, status, sessionStatus, isActive, messageCount24h, lastActivity, errorNote }`
- **SOQL:** `SELECT Id, Display_Name__c, External_Account_ID__c, Channel_Type_API_Name__c, Status__c, Session_Status__c, Active__c, Is_Compromised__c, Last_Connected_At__c, Session_Error_Detail__c, (SELECT Id FROM Messenger_Chats__r LIMIT 1), (SELECT COUNT() FROM Messenger_Messages__r WHERE CreatedDate = LAST_N_DAYS:1) FROM tgint__Messenger_Channel__c WHERE ...dynamic filters... ORDER BY Last_Connected_At__c DESC LIMIT 200`
- **DML:** None
- **Governor impact:** 1 SOQL (with subqueries), 0 DML

#### `getExistingChannel(String channelId)`
- **Parameters:** `channelId` (String, Messenger_Channel__c ID)
- **Return type:** `ChannelDetailWrapper` = full channel + app + config data for wizard pre-population
- **SOQL:** 2 queries (channel + config child)
- **DML:** None
- **Governor impact:** 2 SOQL, 0 DML

#### `createApp(String channelType, Map<String, String> payload)`
- **Parameters:** `channelType` = Channel_Type__mdt DeveloperName, `payload` = `{ appName, apiId, apiHash, ownerEmail }` (varies by type)
- **Return type:** `String` (App record ID)
- **SOQL:** 1 (check for existing app with same API ID to upsert)
- **DML:** 1 upsert on the appropriate `*_App__c` object. Sensitive fields encrypted via `EncryptionService.encrypt()` before DML.
- **Governor impact:** 1 SOQL, 1 DML, 1 Crypto operation

#### `createChannel(String appId, String channelType, Map<String, String> payload)`
- **Parameters:** `appId`, `channelType`, `payload` = `{ channelName, handle, token, assigneeId, assigneeType, autoLinkStrategy }`
- **Return type:** `String` (Channel record ID)
- **SOQL:** 1 (check existing draft channel for upsert)
- **DML:** 2 (insert/upsert `Messenger_Channel__c` + insert/upsert matching `*_Config__c`). Token encrypted via `EncryptionService.encrypt()`.
- **Governor impact:** 1 SOQL, 2 DML, 1 Crypto operation

#### `testConnection(String channelId)`
- **Parameters:** `channelId`
- **Return type:** `ConnectionTestResult` = `{ success: Boolean, botId: String, botUsername: String, details: String, roundTripMs: Integer }`
- **SOQL:** 2 (load channel + config, decrypt token)
- **DML:** 0
- **Callout:** 1 HTTP GET to provider API (e.g., Telegram `getMe`, WhatsApp `GET /v21.0/{phone-number-id}`)
- **Governor impact:** 2 SOQL, 0 DML, 1 callout

#### `activateChannel(String channelId)`
- **Parameters:** `channelId`
- **Return type:** `void`
- **SOQL:** 1
- **DML:** 2 (update `Messenger_Channel__c.Status__c = 'active'`, `Active__c = true` + insert `Channel_Audit_Log__c` with `Event_Type__c = 'channel_activated'`)
- **Governor impact:** 1 SOQL, 2 DML

#### `registerWebhook(String channelId)`
- **Parameters:** `channelId`
- **Return type:** `WebhookRegistrationResult` = `{ success: Boolean, webhookUrl: String, errorMessage: String }`
- **SOQL:** 2 (channel + config)
- **DML:** 2 (update config with webhook secret + insert audit log `Event_Type__c = 'webhook_registered'`)
- **Callout:** 1 HTTP POST to provider API (e.g., Telegram `setWebhook`, Meta Graph API subscription)
- **Governor impact:** 2 SOQL, 2 DML, 1 callout, 1 Crypto operation

#### `toggleActive(String channelId, Boolean isActive)`
- **Parameters:** `channelId`, `isActive`
- **Return type:** `void`
- **DML:** 1 (update `Messenger_Channel__c.Active__c`)
- **Governor impact:** 1 SOQL, 1 DML

#### `softDeleteChannel(String channelId)`
- **Parameters:** `channelId`
- **Return type:** `void`
- **DML:** 2 (update channel `Active__c = false`, `Status__c = 'suspended'` + audit log)
- **Callout:** 1 (unregister webhook with provider)
- **Governor impact:** 1 SOQL, 2 DML, 1 callout

#### `hardDeleteChannel(String channelId)`
- **Parameters:** `channelId`
- **Return type:** `void`
- **SOQL:** 1 (verify zero linked chats)
- **DML:** 3 (delete config + delete channel + audit log)
- **Governor impact:** 1 SOQL, 3 DML

#### `getRecentAuditLogs(String channelId, Integer limitCount)`
- **Parameters:** `channelId` (nullable for all), `limitCount` (default 20)
- **Return type:** `List<AuditLogWrapper>` = `{ timestamp, channelName, eventType, performedBy, source }`
- **SOQL:** `SELECT Event_Timestamp__c, Channel__r.Display_Name__c, Event_Type__c, Performed_By__r.Name, Source__c FROM tgint__Channel_Audit_Log__c WHERE (Channel__c = :channelId OR :channelId = null) ORDER BY Event_Timestamp__c DESC LIMIT :limitCount`
- **Governor impact:** 1 SOQL, 0 DML

### `MessengerController.cls`

#### `getChatList(Map<String, Object> filters, Integer pageSize, Datetime lastMessageCursor)`
- **Parameters:** `filters` = `{ channelIds: List<String>, channelType: String, status: String, searchText: String, unreadOnly: Boolean }`, `pageSize` (default 20), `lastMessageCursor` (nullable, for keyset pagination)
- **Return type:** `ChatListResult` = `{ chats: List<ChatWrapper>, hasMore: Boolean }` where `ChatWrapper` = `{ id, chatTitle, channelId, channelType, channelName, leadId, leadName, contactId, contactName, assignedUserId, assignedUserName, lastMessageAt, lastMessagePreview, unreadCount, status, chatExternalId }`
- **SOQL:** `SELECT Id, Chat_Title__c, Channel__c, Channel__r.Channel_Type_API_Name__c, Channel__r.Display_Name__c, Lead__c, Lead__r.Name, Contact__c, Contact__r.Name, Assigned_User__c, Assigned_User__r.Name, Last_Message_At__c, Last_Message_Preview__c, Unread_Count__c, Status__c, Chat_External_ID__c FROM tgint__Messenger_Chat__c WHERE ...dynamic filters... AND (Last_Message_At__c < :lastMessageCursor OR :lastMessageCursor = null) ORDER BY Last_Message_At__c DESC LIMIT :pageSize+1` -- query pageSize+1 to determine hasMore
- **DML:** None
- **Governor impact:** 1 SOQL, 0 DML. Uses WITH SECURITY_ENFORCED. Sharing rules (Apex Managed Sharing) automatically filter chats the user cannot access.

#### `getChatsForRecord(String recordId)`
- **Parameters:** `recordId` (Lead or Contact ID)
- **Return type:** `List<ChatWrapper>` (same shape as above, filtered by Lead__c or Contact__c = recordId)
- **SOQL:** 1
- **Governor impact:** 1 SOQL

#### `getRecentMessages(String chatId, Datetime sinceTimestamp, Integer pageSize)`
- **Parameters:** `chatId`, `sinceTimestamp` (nullable -- null loads the most recent page), `pageSize` (default 50)
- **Return type:** `MessageListResult` = `{ messages: List<MessageWrapper>, hasMore: Boolean }` where `MessageWrapper` = `{ id, text, direction, messageType, senderName, senderExternalId, deliveryStatus, deliveryError, lastError, sentAt, isEdited, replyToExternalId, mediaUrl, mediaMimeType, templateId, templateName, attachments: List<AttachmentWrapper> }` and `AttachmentWrapper` = `{ id, mediaUrl, mediaMimeType, fileName, fileSize, sortOrder, thumbnailUrl, durationSeconds, width, height }`
- **SOQL:** `SELECT Id, Message_Text__c, Direction__c, Message_Type__c, Sender_Name__c, Sender_External_ID__c, Delivery_Status__c, Delivery_Error__c, Last_Error__c, Sent_At__c, Is_Edited__c, Reply_To_External_ID__c, Media_URL__c, Media_MIME_Type__c, Template__c, Template__r.Template_Name__c, (SELECT Id, Media_URL__c, Media_MIME_Type__c, File_Name__c, File_Size__c, Sort_Order__c, Thumbnail_URL__c, Duration_Seconds__c, Width__c, Height__c FROM Messenger_Attachments__r ORDER BY Sort_Order__c) FROM tgint__Messenger_Message__c WHERE Chat__c = :chatId AND (Sent_At__c < :sinceTimestamp OR :sinceTimestamp = null) ORDER BY Sent_At__c DESC LIMIT :pageSize+1`
- **DML:** None
- **Governor impact:** 1 SOQL (with subquery), 0 DML

#### `sendMessage(String chatId, String text, List<String> attachmentIds)`
- **Parameters:** `chatId`, `text`, `attachmentIds` (ContentVersion IDs, nullable)
- **Return type:** `MessageWrapper` (the newly created message record)
- **Validation:** Verify user has `Channel_User_Access__c` for the chat's channel. Verify chat `Status__c != 'archived'`.
- **SOQL:** 2 (load chat + channel, verify access)
- **DML:** 2 (insert `Messenger_Message__c` + insert `Messenger_Attachment__c` if attachments provided)
- **Async:** Enqueues `MessengerOutboundQueueable` with message ID
- **Governor impact:** 2 SOQL, 2 DML, 1 System.enqueueJob

#### `sendTemplate(String chatId, String templateId, Map<Integer, String> variables)`
- **Parameters:** `chatId`, `templateId` (Messenger_Template__c ID), `variables` (position -> value map)
- **Return type:** `MessageWrapper`
- **Validation:** Same as sendMessage + verify template is approved and active
- **SOQL:** 3 (chat + channel + template)
- **DML:** 1 (insert `Messenger_Message__c` with `Message_Type__c = 'template'`, `Template__c = templateId`)
- **Async:** Enqueues `MessengerOutboundQueueable`
- **Governor impact:** 3 SOQL, 1 DML, 1 System.enqueueJob

#### `markAsRead(String chatId)`
- **Parameters:** `chatId`
- **Return type:** `void`
- **DML:** 1 (update `Messenger_Chat__c.Unread_Count__c = 0`)
- **Governor impact:** 1 SOQL, 1 DML

#### `linkToRecord(String chatId, String recordId, String recordType)`
- **Parameters:** `chatId`, `recordId` (Lead or Contact ID), `recordType` ('Lead' or 'Contact')
- **Return type:** `void`
- **DML:** 1 (update `Messenger_Chat__c.Lead__c` or `Contact__c`)
- **Governor impact:** 1 SOQL, 1 DML

#### `getTemplates(String channelId)`
- **Parameters:** `channelId`
- **Return type:** `List<TemplateWrapper>` = `{ id, name, language, body, header, footer, category, variableSchema, approvalStatus }`
- **SOQL:** `SELECT Id, Template_Name__c, Template_Language__c, Template_Body__c, Template_Header__c, Template_Footer__c, Template_Category__c, Variable_Schema_JSON__c, Approval_Status__c FROM tgint__Messenger_Template__c WHERE (Channel__c = :channelId OR Channel__c = null) AND Approval_Status__c = 'approved' AND Is_Active__c = true ORDER BY Template_Name__c`
- **Governor impact:** 1 SOQL, 0 DML

#### `getInboxChats(String tabFilter)`
- **Parameters:** `tabFilter` ('new', 'live', 'pending')
- **Return type:** `List<InboxChatWrapper>` = `{ chatId, chatTitle, channelName, channelType, lastMessagePreview, lastMessageDirection, unreadCount, leadId, leadName }`
- **SOQL:** Dynamic based on tab: 'new' = `Unread_Count__c > 0`, 'live' = last message direction = 'outbound', 'pending' = last message direction = 'inbound'. Limit 30.
- **Governor impact:** 1 SOQL, 0 DML

#### `sendQuickReply(String chatId, String text)`
- **Parameters:** `chatId`, `text`
- **Return type:** `MessageWrapper`
- **Notes:** Delegates to `sendMessage(chatId, text, null)` internally
- **Governor impact:** Same as sendMessage

---

## 4. Platform Event Subscriptions

### `messengerChat` (full mode and embedded mode)

| Event Channel | Filter Criteria | UI Update | Edge Cases |
|---|---|---|---|
| `/event/tgint__Inbound_Message__e` | `Chat_SF_ID__c` matches any chat in the current view (active chat ID for thread, all visible chat IDs for list) | **Active chat:** append message to thread, scroll to bottom. **Chat list:** update `lastMessagePreview`, increment unread badge, re-sort list. | Component not visible: PE is received but UI update deferred until `renderedCallback`. CometD reconnect: on reconnect, call `getRecentMessages` to fill any gaps. |
| `/event/tgint__Message_Delivery_Status__e` | `Message_SF_ID__c` matches any pending outbound message in the current thread | Update the delivery status icon on the matching bubble: pending -> sent -> delivered -> read -> failed. | PE arrives before sendMessage returns: buffer in a `pendingStatusUpdates` map, replay after the imperative response. |

### `messengerInbox` (utility bar)

| Event Channel | Filter Criteria | UI Update | Edge Cases |
|---|---|---|---|
| `/event/tgint__Inbound_Message__e` | All events (no filter -- inbox shows all chats) | Increment the badge count on the utility bar "Messages" button. If popup is open, prepend the chat to the "New Chats" tab. | Component always mounted in utility bar, so subscription is persistent. On CometD reconnect, call `getInboxChats('new')` to resync badge count. |

### `messengerChannelList`

No PE subscriptions. Channel status changes are not frequent enough to warrant real-time updates. The list refreshes on tab activation and after wizard actions.

### Subscription lifecycle

All components subscribe in `connectedCallback()` via `import { subscribe, unsubscribe, onError } from 'lightning/empApi'`. They call `unsubscribe()` in `disconnectedCallback()`. The `onError` handler logs to console and attempts resubscription after a 3-second delay, with a maximum of 3 retry attempts before showing a toast "Real-time updates temporarily unavailable. Please refresh the page."

---

## 5. Implementation Plan

### Stream A -- Data Model

| # | Task | Complexity | Deliverable |
|---|------|-----------|-------------|
| A1 | Create all 21 custom objects with fields per sf-data-model.md | L | Object XML files |
| A2 | Create 3 platform event definitions with fields | S | PE XML files |
| A3 | Create Channel_Type__mdt with 9 seed records (6 Phase 1 active, 3 Phase 2 disabled) | M | CMT XML + record files |
| A4 | Create Encryption_Key__mdt definition (Protected, no seed data -- populated at install time) | S | CMT XML file |
| A5 | Create Messenger_Settings__mdt definition + default record | S | CMT XML + record file |
| A6 | Create 3 Permission Sets (Messenger_Admin, Messenger_Agent, Messenger_Viewer) with object/field permissions | M | Permission Set XML files |
| A7 | Create CustomNotificationType for Messenger_New_Message | S | Notification type XML |
| A8 | Add `Message_SF_ID__c` field to `Message_Delivery_Status__e` | S | PE field XML |
| A9 | Add `Variable_Schema_JSON__c` field to `Messenger_Template__c` | S | Field XML |

**Dependencies:** None -- this stream runs first.
**Verification:** `sf project deploy start --source-dir MessageForge.SalesForce/force-app --dry-run` passes with zero errors.

### Stream B -- Channel Setup Wizard

| # | Task | Complexity | Deliverable |
|---|------|-----------|-------------|
| B1 | Implement `EncryptionService.cls` (AES-256 encrypt/decrypt using Encryption_Key__mdt) | M | Apex class |
| B2 | Implement `ChannelSetupController.cls` with all 11 @AuraEnabled methods | L | Apex class |
| B3 | Implement `channelSetupWizard` LWC + 5 child step components | L | 6 LWC component folders |
| B4 | Implement `messengerChannelList` LWC with filters, actions, audit log | M | 1 LWC component folder |
| B5 | Implement `channelAuditLog` LWC | S | 1 LWC component folder |
| B6 | Create per-provider webhook registration callout classes (TelegramWebhookService, WhatsAppWebhookService, etc.) | M | 6 Apex classes |

**Dependencies:** Stream A (objects and CMTs must exist).
**Verification:** Deploy validates. Wizard LWC compiles. Controller methods are callable from Developer Console (manual test with mock data).

### Stream C -- Inbound Handlers

| # | Task | Complexity | Deliverable |
|---|------|-----------|-------------|
| C1 | Implement `MessengerInboundAPI.cls` (@RestResource) with generic skeleton | M | Apex class |
| C2 | Implement `HMACValidator.cls` with per-channel verification strategies | M | Apex class |
| C3 | Implement 6 inbound parser classes (TelegramInboundParser, WhatsAppInboundParser, ViberInboundParser, TwilioInboundParser, MessengerInboundParser, InstagramInboundParser) | L | 6 Apex classes + 1 interface |
| C4 | Implement `NormalizedInboundDTO` and `NormalizedAttachmentDTO` classes | S | 2 Apex classes |
| C5 | Implement `InboundMessageTrigger` (after insert on Messenger_Message__c: publish Inbound_Message__e, send custom notification) | M | 1 trigger + 1 trigger handler class |
| C6 | Implement media download logic (sync for <=5MB, Queueable for 5-12MB, graceful fail for >12MB) | M | 1 Apex class (MediaDownloadService) + 1 Queueable |

**Dependencies:** Stream A (objects, PEs). Stream B partially (EncryptionService for token decryption, but can use a mock/stub).
**Verification:** Deploy validates. REST endpoint responds to POST with mock payloads.

### Stream D -- Outbound Flow

| # | Task | Complexity | Deliverable |
|---|------|-----------|-------------|
| D1 | Implement `MessengerOutboundService.cls` (dispatcher that routes to per-channel senders) | M | Apex class |
| D2 | Implement `MessengerOutboundQueueable.cls` with retry logic (max 3 attempts, backoff) | M | Apex class |
| D3 | Implement 6 per-channel sender classes (TelegramSender, WhatsAppSender, ViberSender, TwilioSender, MessengerSender, InstagramSender) | L | 6 Apex classes + 1 interface |
| D4 | Implement `DeliveryStatusTrigger` (after update on Messenger_Message__c: publish Message_Delivery_Status__e when Delivery_Status__c changes) | S | 1 trigger + 1 handler class |
| D5 | Create Named Credential metadata templates for each provider | M | Named Credential XML files |

**Dependencies:** Stream A (objects, PEs). Stream B (EncryptionService).
**Verification:** Deploy validates. Queueable compiles.

### Stream E -- LWC Chat UI

| # | Task | Complexity | Deliverable |
|---|------|-----------|-------------|
| E1 | Implement `MessengerController.cls` with all @AuraEnabled methods | L | Apex class |
| E2 | Implement `messengerChat` container LWC (full + embedded modes) with PE subscriptions | L | 1 LWC component folder |
| E3 | Implement `messengerChatBubble` LWC (all message types + error states) | M | 1 LWC component folder |
| E4 | Implement `messengerChannelFilter` LWC | S | 1 LWC component folder |
| E5 | Implement `messengerComposer` LWC (textarea + file upload + send) | M | 1 LWC component folder |
| E6 | Implement `messengerTemplatePicker` LWC | M | 1 LWC component folder |
| E7 | Implement `messengerMediaViewer` LWC | S | 1 LWC component folder |
| E8 | Implement `messengerInbox` LWC (utility bar popup) | M | 1 LWC component folder |
| E9 | Implement `messengerRecordLinker` LWC | S | 1 LWC component folder |
| E10 | Implement `messengerChatList` and `messengerChatThread` internal components | M | 2 LWC component folders |

**Dependencies:** Stream A (objects, PEs). Stream C and D (for end-to-end message flow, but LWCs can be built against mock data first).
**Verification:** Deploy validates. All LWCs compile. Components render in App Builder preview.

### Stream F -- Security & Sharing

| # | Task | Complexity | Deliverable |
|---|------|-----------|-------------|
| F1 | Implement `ChannelAccessService.cls` (Apex Managed Sharing recalculation) | M | Apex class |
| F2 | Implement `ChannelAccessTrigger` (on Channel_User_Access__c insert/update/delete: recalc sharing) | M | 1 trigger + 1 handler class |
| F3 | Implement `ChannelCompromiseTrigger` (on Messenger_Channel__c update: when Is_Compromised__c = true, log + block -- Phase 1 stub for Go notification) | S | 1 trigger + 1 handler class |
| F4 | Implement audit log insert utility (ChannelAuditService.cls) | S | Apex class |
| F5 | Create Apex Managed Sharing objects (Messenger_Chat__Share) | S | Object XML |
| F6 | Review and harden EncryptionService.cls (key rotation support, error handling) | S | Updates to existing class |

**Dependencies:** Stream A (objects). Stream B (EncryptionService).
**Verification:** Deploy validates. Sharing records created correctly when Channel_User_Access__c rows are inserted.

### Stream G -- Tests

| # | Task | Complexity | Deliverable |
|---|------|-----------|-------------|
| G1 | Test data factory class (TestDataFactory.cls) | M | Apex class |
| G2 | Test classes for Stream B (ChannelSetupControllerTest, EncryptionServiceTest) | L | 2+ test classes |
| G3 | Test classes for Stream C (MessengerInboundAPITest, HMACValidatorTest, per-parser tests) | L | 8+ test classes |
| G4 | Test classes for Stream D (MessengerOutboundServiceTest, OutboundQueueableTest, per-sender tests) | L | 8+ test classes |
| G5 | Test classes for Stream E (MessengerControllerTest) | L | 1+ test classes |
| G6 | Test classes for Stream F (ChannelAccessServiceTest, trigger tests) | M | 3+ test classes |
| G7 | Mock callout classes (HttpCalloutMock implementations for each provider) | M | 6+ mock classes |
| G8 | Bulk test scenarios (200 records per trigger test) | M | Included in above test classes |
| G9 | Negative test cases (invalid HMAC, missing fields, unauthorized access) | M | Included in above test classes |

**Dependencies:** All other streams (B through F).
**Verification:** `sf apex run test --test-level RunLocalTests --code-coverage --wait 30` reports 80%+ coverage for every class.

### Dependency Graph

```
A (Data Model) ─┬──► B (Channel Setup Wizard) ──► G (Tests)
                 ├──► C (Inbound Handlers) ───────► G
                 ├──► D (Outbound Flow) ──────────► G
                 ├──► E (LWC Chat UI) ────────────► G
                 └──► F (Security & Sharing) ─────► G
                       B ──► C (EncryptionService dependency)
                       B ──► D (EncryptionService dependency)
                       C + D ──► E (end-to-end flow, but E can start with mocks)
```

Streams B, C, D, E, F can run in parallel after A completes. Stream G runs last.

---

## 6. Data Model Deltas

Changes beyond what is documented in `sf-data-model.md`:

### New fields

| Object | Field | Type | Purpose |
|--------|-------|------|---------|
| `tgint__Message_Delivery_Status__e` | `Message_SF_ID__c` | Text(18) | Salesforce record ID of the Messenger_Message__c so LWC can correlate PE to optimistically-rendered outbound messages |
| `tgint__Messenger_Template__c` | `Variable_Schema_JSON__c` | Long Text Area(5000) | JSON array describing template variable positions, labels, and examples for the template picker UI |

### New metadata

| Type | Name | Purpose |
|------|------|---------|
| `CustomNotificationType` | `Messenger_New_Message` | Bell notification for assigned user on new inbound message |

### New CMT seed records for `Channel_Type__mdt`

| DeveloperName | Messenger__c | Protocol__c | Config_Object_Name__c | Supports_Outbound__c | Phase 1 Active |
|---|---|---|---|---|---|
| `Telegram_Bot` | Telegram | bot_api | Telegram_Bot_Config__c | true | Yes |
| `WhatsApp_Cloud` | WhatsApp | cloud_api | WhatsApp_Config__c | true | Yes |
| `Viber_Bot` | Viber | bot_api | Viber_Config__c | true | Yes |
| `Twilio_SMS` | Twilio | rest | Twilio_Config__c | true | Yes |
| `Facebook_Messenger` | Facebook | cloud_api | (uses WhatsApp_App__c for Meta credentials) | true | Yes |
| `Instagram_Direct` | Instagram | cloud_api | (uses WhatsApp_App__c for Meta credentials) | true | Yes |
| `Telegram_User` | Telegram | mtproto | Telegram_User_Config__c | true | No (Phase 2) |
| `Apple_Messages` | Apple | msp_api | Apple_Messages_Config__c | true | No (Phase 2) |
| `Vonage_SMS` | Vonage | rest | Vonage_Config__c | true | No (Phase 2) |

Note: Facebook Messenger and Instagram Direct share `WhatsApp_App__c` for Meta App credentials (same App ID / App Secret). They need a new config object or reuse mechanism. For Phase 1, create two lightweight config objects:

| New Object | Fields | Relationship |
|---|---|---|
| `tgint__Messenger_Config__c` | Page_ID__c, Page_Access_Token__c (encrypted), Verify_Token__c | Master-Detail to Messenger_Channel__c |
| `tgint__Instagram_Config__c` | Instagram_Business_ID__c, Page_ID__c, Page_Access_Token__c (encrypted) | Master-Detail to Messenger_Channel__c |

These are additions to the schema -- the existing `sf-data-model.md` does not include config objects for Facebook Messenger or Instagram Direct separately.

---

## Risks & dependencies

- **Governor limit risk on Platform Events:** The 10K/day delivery limit for Enterprise edition is a hard business constraint. Phase 1 documents this and recommends Unlimited edition for customers with >50 concurrent agents.
- **HMAC verification correctness:** Each provider uses different HMAC algorithms and header names. Exhaustive integration testing (Stream G) with recorded provider payloads is critical.
- **AppExchange Security Review:** All callouts must use Named Credentials. No hardcoded URLs. All user input sanitized. EncryptionService must pass the Checkmarx static analysis. Plan for 2-4 weeks of security review iteration after dev complete.
- **Media heap limit:** The 12 MB async heap limit constrains file downloads. The graceful degradation path (`media_too_large_phase2`) must be tested for every provider.
- **Named Credential creation at runtime:** Salesforce requires Connected App + External Credential + Named Credential metadata. Creating these dynamically from Apex is limited -- may need to use Metadata API deployments from Apex, which adds complexity.
- **WhatsApp 24-hour session window:** The LWC must detect when a chat is outside the 24h window and force the template picker. This requires tracking the last inbound message timestamp per chat.

## Test strategy

- **Unit tests (Stream G):** 80%+ line coverage for every Apex class. Bulk tests with 200 records per trigger test.
- **Mock callouts:** `HttpCalloutMock` implementations for each provider, returning realistic response bodies.
- **Negative tests:** Invalid HMAC, missing required fields, unauthorized access attempts, duplicate message dedup, media too large.
- **Integration verification:** Deploy to scratch org, create a test channel, send/receive messages end-to-end with a real provider (Telegram Bot is easiest to test).
- **LWC testing:** Jest unit tests for each component (mocking Apex wire adapters and imperative calls).
- **Security Review prep:** Run Checkmarx locally, verify no hardcoded secrets, verify all SOQL uses bind variables, verify all DML respects sharing.

---

## Agent Prompts

### Stream A -- Data Model

You are a Salesforce metadata engineer working on the MessageForge managed package. Your task is to create all custom object definitions, custom metadata types with seed records, platform events, permission sets, and a custom notification type for the Phase 1 data model.

**Goal:** Deploy the complete Phase 1 data model as SFDX source metadata so that all subsequent development streams (Apex controllers, LWCs, triggers) can compile and deploy against it.

**Files to read first:**
- `MessageForge.SalesForce/CLAUDE.md` -- subsystem conventions
- `MessageForge.SalesForce/sfdx-project.json` -- project config
- `MessageForge.Documentation/handoff/sf-data-model.md` -- canonical data model reference

**Requires:** None -- this stream runs first.

**Specification:**

All API names use the namespace prefix `tgint__`. All metadata uses API version `62.0`. SFDX source format. Target directory: `MessageForge.SalesForce/force-app/main/default/`.

**Custom Objects to create (with all fields as specified below):**

1. **`tgint__Messenger_Channel__c`**
   - `Display_Name__c` -- Text(255), Required
   - `Channel_Type_API_Name__c` -- Text(80), Required (DeveloperName of Channel_Type__mdt)
   - `Status__c` -- Picklist: draft, active, suspended, archived. Default: draft
   - `Session_Status__c` -- Picklist: online, offline, connecting, auth_required, mfa_required, flood_wait, error. Default: offline
   - `Session_Error_Detail__c` -- Long Text Area(2000)
   - `Session_Last_Heartbeat__c` -- DateTime
   - `Active__c` -- Checkbox, Default: false
   - `Is_Compromised__c` -- Checkbox, Default: false
   - `External_Account_ID__c` -- Text(255)
   - `Go_Connection_Ref__c` -- Text(255)
   - `Last_Connected_At__c` -- DateTime
   - `Credential_Rotated_At__c` -- DateTime

2. **`tgint__Telegram_App__c`**
   - `App_Name__c` -- Text(255), Required
   - `API_ID__c` -- Text(20), Required
   - `API_Hash__c` -- Long Text Area(1000) (encrypted)
   - `Owner_Email__c` -- Email
   - `Manager__c` -- Lookup(User)

3. **`tgint__WhatsApp_App__c`**
   - `App_Name__c` -- Text(255), Required
   - `App_ID__c` -- Text(50), Required
   - `App_Secret__c` -- Long Text Area(1000) (encrypted)
   - `Owner_Email__c` -- Email
   - `Manager__c` -- Lookup(User)

4. **`tgint__Twilio_App__c`**
   - `App_Name__c` -- Text(255), Required
   - `Account_SID__c` -- Text(50), Required
   - `Auth_Token__c` -- Long Text Area(1000) (encrypted)
   - `Owner_Email__c` -- Email
   - `Manager__c` -- Lookup(User)

5. **`tgint__Vonage_App__c`** (Phase 2 schema, deploy but unused)
   - `App_Name__c` -- Text(255), Required
   - `API_Key__c` -- Text(50)
   - `API_Secret__c` -- Long Text Area(1000) (encrypted)
   - `Manager__c` -- Lookup(User)

6. **`tgint__Bird_App__c`** (Phase 2 schema, deploy but unused)
   - `App_Name__c` -- Text(255), Required
   - `Access_Key__c` -- Long Text Area(1000) (encrypted)
   - `Organization_ID__c` -- Text(100)
   - `Manager__c` -- Lookup(User)

7. **`tgint__Telegram_Bot_Config__c`**
   - `Channel__c` -- Master-Detail(Messenger_Channel__c)
   - `Bot_Token__c` -- Long Text Area(1000) (encrypted)
   - `Bot_Username__c` -- Text(100)
   - `App__c` -- Lookup(Telegram_App__c)
   - `Webhook_Secret__c` -- Long Text Area(500) (encrypted)
   - `Allowed_Updates__c` -- Text(500) (comma-separated: message, edited_message, callback_query)

8. **`tgint__Telegram_User_Config__c`** (Phase 2, deploy but unused)
   - `Channel__c` -- Master-Detail(Messenger_Channel__c)
   - `Phone__c` -- Phone
   - `Two_FA_Password__c` -- Long Text Area(500) (encrypted)
   - `App__c` -- Lookup(Telegram_App__c)

9. **`tgint__WhatsApp_Config__c`**
   - `Channel__c` -- Master-Detail(Messenger_Channel__c)
   - `Phone_Number_ID__c` -- Text(50), Required
   - `WABA_ID__c` -- Text(50), Required
   - `Access_Token__c` -- Long Text Area(1000) (encrypted)
   - `App__c` -- Lookup(WhatsApp_App__c)
   - `Verify_Token__c` -- Text(255)
   - `Webhook_Secret__c` -- Long Text Area(500) (encrypted)

10. **`tgint__Viber_Config__c`**
    - `Channel__c` -- Master-Detail(Messenger_Channel__c)
    - `Auth_Token__c` -- Long Text Area(1000) (encrypted)
    - `Bot_Name__c` -- Text(100)
    - `Webhook_Secret__c` -- Long Text Area(500) (encrypted)

11. **`tgint__Twilio_Config__c`**
    - `Channel__c` -- Master-Detail(Messenger_Channel__c)
    - `Phone__c` -- Phone, Required
    - `API_Key__c` -- Text(100)
    - `Messaging_Service_SID__c` -- Text(50)
    - `App__c` -- Lookup(Twilio_App__c)
    - `Webhook_Secret__c` -- Long Text Area(500) (encrypted)

12. **`tgint__Vonage_Config__c`** (Phase 2)
    - `Channel__c` -- Master-Detail(Messenger_Channel__c)
    - `Phone__c` -- Phone
    - `Application_ID__c` -- Text(100)
    - `Private_Key__c` -- Long Text Area(5000) (encrypted)
    - `App__c` -- Lookup(Vonage_App__c)

13. **`tgint__Bird_Config__c`** (Phase 2)
    - `Channel__c` -- Master-Detail(Messenger_Channel__c)
    - `Phone__c` -- Phone
    - `Access_Key__c` -- Long Text Area(1000) (encrypted)
    - `Workspace_ID__c` -- Text(100)
    - `Channel_ID__c` -- Text(100)
    - `App__c` -- Lookup(Bird_App__c)

14. **`tgint__Apple_Messages_Config__c`** (Phase 2)
    - `Channel__c` -- Master-Detail(Messenger_Channel__c)
    - `Business_ID__c` -- Text(100)
    - `Intent_ID__c` -- Text(100)
    - `Registration_Type__c` -- Picklist: standard, custom

15. **`tgint__Messenger_Config__c`** (NEW -- Facebook Messenger config)
    - `Channel__c` -- Master-Detail(Messenger_Channel__c)
    - `Page_ID__c` -- Text(50), Required
    - `Page_Access_Token__c` -- Long Text Area(1000) (encrypted)
    - `Verify_Token__c` -- Text(255)
    - `App__c` -- Lookup(WhatsApp_App__c) (shared Meta App)
    - `Webhook_Secret__c` -- Long Text Area(500) (encrypted)

16. **`tgint__Instagram_Config__c`** (NEW -- Instagram Direct config)
    - `Channel__c` -- Master-Detail(Messenger_Channel__c)
    - `Instagram_Business_ID__c` -- Text(100), Required
    - `Page_ID__c` -- Text(50), Required
    - `Page_Access_Token__c` -- Long Text Area(1000) (encrypted)
    - `App__c` -- Lookup(WhatsApp_App__c) (shared Meta App)
    - `Verify_Token__c` -- Text(255)
    - `Webhook_Secret__c` -- Long Text Area(500) (encrypted)

17. **`tgint__Channel_User_Access__c`**
    - `Channel__c` -- Master-Detail(Messenger_Channel__c)
    - `Assignee_Type__c` -- Picklist: user, group. Required
    - `User__c` -- Lookup(User)
    - `Group_ID__c` -- Text(18)
    - `Group_Name__c` -- Text(255)
    - `Access_Level__c` -- Picklist: owner, read_write, read_only. Default: read_write
    - `Is_Primary_Assignee__c` -- Checkbox, Default: false
    - `Is_Active__c` -- Checkbox, Default: true

18. **`tgint__Messenger_Chat__c`**
    - `Channel__c` -- Lookup(Messenger_Channel__c) (not MD -- survives channel deletion)
    - `Lead__c` -- Lookup(Lead)
    - `Contact__c` -- Lookup(Contact)
    - `Assigned_User__c` -- Lookup(User)
    - `Chat_External_ID__c` -- Text(255), External ID, Unique
    - `Chat_Title__c` -- Text(255)
    - `Chat_Type__c` -- Picklist: private, group, supergroup, channel. Default: private
    - `Status__c` -- Picklist: open, pending, resolved, archived. Default: open
    - `Last_Message_At__c` -- DateTime (indexed)
    - `Last_Message_Preview__c` -- Text(255)
    - `Unread_Count__c` -- Number(5,0), Default: 0

19. **`tgint__Messenger_Message__c`**
    - `Chat__c` -- Master-Detail(Messenger_Chat__c)
    - `Message_External_ID__c` -- Text(255), External ID, Unique
    - `Sender_Name__c` -- Text(255)
    - `Sender_External_ID__c` -- Text(255)
    - `Message_Text__c` -- Long Text Area(32000)
    - `Direction__c` -- Picklist: inbound, outbound. Required
    - `Message_Type__c` -- Picklist: text, photo, video, voice, document, sticker, template, album, system
    - `Template__c` -- Lookup(Messenger_Template__c)
    - `Media_URL__c` -- URL
    - `Media_MIME_Type__c` -- Text(100)
    - `Attachment_Count__c` -- Number(3,0) (Roll-Up Summary: COUNT of Messenger_Attachment__c)
    - `Delivery_Status__c` -- Picklist: pending, sent, delivered, read, failed. Default: pending
    - `Delivery_Error__c` -- Text(500)
    - `Last_Error__c` -- Text(500)
    - `Sent_At__c` -- DateTime
    - `Reply_To_External_ID__c` -- Text(255)
    - `Is_Edited__c` -- Checkbox, Default: false

20. **`tgint__Messenger_Attachment__c`**
    - `Message__c` -- Master-Detail(Messenger_Message__c)
    - `Media_URL__c` -- URL
    - `Media_MIME_Type__c` -- Text(100)
    - `File_Name__c` -- Text(255)
    - `File_Size__c` -- Number(10,0)
    - `Sort_Order__c` -- Number(3,0)
    - `Thumbnail_URL__c` -- URL
    - `Media_External_ID__c` -- Text(255)
    - `Duration_Seconds__c` -- Number(6,0)
    - `Width__c` -- Number(5,0)
    - `Height__c` -- Number(5,0)

21. **`tgint__Messenger_Template__c`**
    - `Channel__c` -- Lookup(Messenger_Channel__c)
    - `Template_Name__c` -- Text(255), Required
    - `Template_Language__c` -- Text(10) (BCP 47)
    - `Template_Body__c` -- Long Text Area(10000)
    - `Template_Header__c` -- Text(1000)
    - `Template_Footer__c` -- Text(500)
    - `Template_Category__c` -- Picklist: marketing, utility, authentication
    - `Variable_Schema_JSON__c` -- Long Text Area(5000) (NEW)
    - `Platform_Template_ID__c` -- Text(100)
    - `Approval_Status__c` -- Picklist: draft, pending, approved, rejected, paused, disabled. Default: draft
    - `Rejection_Reason__c` -- Text(1000)
    - `Is_Active__c` -- Checkbox, Default: false
    - `Messenger__c` -- Picklist: WhatsApp, Viber, Twilio, Vonage, Bird
    - `Button_JSON__c` -- Long Text Area(5000)
    - `Last_Synced_At__c` -- DateTime
    - `Usage_Count__c` -- Number(8,0), Default: 0

22. **`tgint__Channel_Audit_Log__c`**
    - `Channel__c` -- Lookup(Messenger_Channel__c) (survives deletion)
    - `Event_Type__c` -- Picklist: credential_rotated, credential_expired, compromised, session_connected, session_disconnected, channel_activated, channel_suspended, webhook_registered, webhook_unregistered, hmac_verification_failed, manual_link_changed, sharing_recalculated
    - `Event_Detail__c` -- Long Text Area(5000) (JSON)
    - `Performed_By__c` -- Lookup(User)
    - `Event_Timestamp__c` -- DateTime, Required
    - `Source__c` -- Picklist: salesforce, go_middleware. Default: salesforce

23. **`tgint__Messenger_Event__c`** (deploy but unused in Phase 1)
    - `Message__c` -- Lookup(Messenger_Message__c)
    - `Event_Title__c` -- Text(255)
    - `Event_Date__c` -- DateTime
    - `Event_Location__c` -- Text(500)
    - `Event_Description__c` -- Long Text Area(5000)
    - `Processing_Status__c` -- Picklist: pending, confirmed, discarded. Default: pending

**Platform Events:**

1. **`tgint__Inbound_Message__e`**
   - `Text__c` -- Long Text Area(5000)
   - `Chat_SF_ID__c` -- Text(18)
   - `Chat_External_ID__c` -- Text(255)
   - `Sender_Name__c` -- Text(255)
   - `Sender_External_ID__c` -- Text(255)
   - `Message_Type__c` -- Text(50)
   - `Media_URL__c` -- URL
   - `Media_MIME_Type__c` -- Text(100)
   - `Protocol__c` -- Text(50)
   - `Sent_At__c` -- DateTime
   - `Message_SF_ID__c` -- Text(18) (the Salesforce record ID of the inserted message)
   - Publish Behavior: Publish After Commit

2. **`tgint__Session_Status__e`** (deploy, unused in Phase 1)
   - `Connection_SF_ID__c` -- Text(18)
   - `Status__c` -- Text(50)
   - `Error_Detail__c` -- Text(500)
   - Publish Behavior: Publish After Commit

3. **`tgint__Message_Delivery_Status__e`**
   - `Message_External_ID__c` -- Text(255)
   - `Message_SF_ID__c` -- Text(18) (NEW: Salesforce record ID for LWC correlation)
   - `Status__c` -- Text(50)
   - `Error_Code__c` -- Text(100)
   - `Error_Detail__c` -- Text(500)
   - `Timestamp__c` -- DateTime
   - Publish Behavior: Publish After Commit

**Custom Metadata Types:**

1. **`tgint__Channel_Type__mdt`** -- NOT Protected (visible to customer for reference)
   - `Messenger__c` -- Text(50)
   - `Protocol__c` -- Text(50)
   - `Config_Object_Name__c` -- Text(100)
   - `Supports_Outbound__c` -- Checkbox
   - Seed 9 records as specified in the Data Model Deltas section above.

2. **`tgint__Encryption_Key__mdt`** -- Protected
   - `Key_Value__c` -- Long Text Area(500)
   - `Is_Active__c` -- Checkbox, Default: true
   - No seed records (populated at install time).

3. **`tgint__Messenger_Settings__mdt`** -- Protected
   - `HMAC_Secret__c` -- Long Text Area(500)
   - `Go_Middleware_URL__c` -- URL
   - `Rate_Limit_Bucket_Size__c` -- Number(5,0)
   - `Is_Active__c` -- Checkbox, Default: true
   - Seed 1 default record with `Is_Active__c = true`, other fields blank.

4. **`tgint__Middleware_Config__mdt`** -- Protected (Phase 2, deploy definition only)
   - `Instance_Region__c` -- Text(50)
   - `Max_Concurrent_Workers__c` -- Number(5,0)
   - `Batching_Interval_Seconds__c` -- Number(3,0)
   - `Max_Batch_Size_KB__c` -- Number(6,0)

**Permission Sets:**

1. **`tgint__Messenger_Admin`** -- Full CRUD on all tgint__ objects. Read on CMTs. Publish platform events. Access to all Apex classes and LWC components.

2. **`tgint__Messenger_Agent`** -- Read on Messenger_Channel__c. CRUD on Messenger_Chat__c, Messenger_Message__c, Messenger_Attachment__c. Read on Messenger_Template__c. Create on Channel_Audit_Log__c. Access to MessengerController, messengerChat, messengerInbox, messengerMediaViewer LWCs.

3. **`tgint__Messenger_Viewer`** -- Read-only on Messenger_Channel__c, Messenger_Chat__c, Messenger_Message__c, Messenger_Attachment__c, Channel_Audit_Log__c. Access to messengerChat (read-only mode), messengerMediaViewer.

**Custom Notification Type:**

- `Messenger_New_Message` -- send notification, target `Standard__LightningPage` (navigates to Chat record page).

**Deliverables (exact file paths relative to `MessageForge.SalesForce/`):**

```
force-app/main/default/objects/tgint__Messenger_Channel__c/
force-app/main/default/objects/tgint__Telegram_App__c/
force-app/main/default/objects/tgint__WhatsApp_App__c/
force-app/main/default/objects/tgint__Twilio_App__c/
force-app/main/default/objects/tgint__Vonage_App__c/
force-app/main/default/objects/tgint__Bird_App__c/
force-app/main/default/objects/tgint__Telegram_Bot_Config__c/
force-app/main/default/objects/tgint__Telegram_User_Config__c/
force-app/main/default/objects/tgint__WhatsApp_Config__c/
force-app/main/default/objects/tgint__Viber_Config__c/
force-app/main/default/objects/tgint__Twilio_Config__c/
force-app/main/default/objects/tgint__Vonage_Config__c/
force-app/main/default/objects/tgint__Bird_Config__c/
force-app/main/default/objects/tgint__Apple_Messages_Config__c/
force-app/main/default/objects/tgint__Messenger_Config__c/
force-app/main/default/objects/tgint__Instagram_Config__c/
force-app/main/default/objects/tgint__Channel_User_Access__c/
force-app/main/default/objects/tgint__Messenger_Chat__c/
force-app/main/default/objects/tgint__Messenger_Message__c/
force-app/main/default/objects/tgint__Messenger_Attachment__c/
force-app/main/default/objects/tgint__Messenger_Template__c/
force-app/main/default/objects/tgint__Channel_Audit_Log__c/
force-app/main/default/objects/tgint__Messenger_Event__c/
force-app/main/default/platformEvents/tgint__Inbound_Message__e.platformEvent-meta.xml
force-app/main/default/platformEvents/tgint__Message_Delivery_Status__e.platformEvent-meta.xml
force-app/main/default/platformEvents/tgint__Session_Status__e.platformEvent-meta.xml
force-app/main/default/customMetadata/tgint__Channel_Type__mdt.*
force-app/main/default/customMetadata/tgint__Encryption_Key__mdt (definition only)
force-app/main/default/customMetadata/tgint__Messenger_Settings__mdt.*
force-app/main/default/customMetadata/tgint__Middleware_Config__mdt (definition only)
force-app/main/default/permissionsets/tgint__Messenger_Admin.permissionset-meta.xml
force-app/main/default/permissionsets/tgint__Messenger_Agent.permissionset-meta.xml
force-app/main/default/permissionsets/tgint__Messenger_Viewer.permissionset-meta.xml
force-app/main/default/notificationtypes/Messenger_New_Message.notiftype-meta.xml
```

Each object directory must contain an `*.object-meta.xml` and a `fields/` subdirectory with individual `*.field-meta.xml` files per SFDX source format conventions.

**Acceptance criteria:**
1. `sf project deploy start --source-dir MessageForge.SalesForce/force-app --dry-run` passes with zero errors.
2. All 23 objects have their correct fields, relationship types, and picklist values.
3. All 3 platform events have their correct fields.
4. Channel_Type__mdt has 9 seed records.
5. Messenger_Settings__mdt has 1 default record.
6. All 3 permission sets are syntactically valid.

**Constraints:**
- Namespace prefix: `tgint__` for all custom objects/fields/events
- API version: `62.0`
- Package type: 2GP managed package
- Format: SFDX source format
- Target directory: `MessageForge.SalesForce/force-app/main/default/`
- Do NOT delete the existing `.gitkeep` file -- leave it in place

---

### Stream B -- Channel Setup Wizard

You are a Salesforce Apex and LWC developer working on the MessageForge managed package. Your task is to build the channel setup wizard UI and its Apex backend, enabling admins to create, configure, test, and activate messenger channels.

**Goal:** Deliver the `channelSetupWizard` LWC, `messengerChannelList` LWC, `channelAuditLog` LWC, `ChannelSetupController.cls`, `EncryptionService.cls`, and per-provider webhook registration service classes.

**Files to read first:**
- `MessageForge.SalesForce/CLAUDE.md` -- subsystem conventions
- `MessageForge.SalesForce/sfdx-project.json` -- project config
- `MessageForge.Documentation/handoff/messageforge-phase1-technical-handoff.md` -- full architecture
- `MessageForge.Documentation/handoff/sf-data-model.md` -- data model
- `MessageForge.Documentation/handoff/PROTOTYPE-HANDOFF.md` -- prototype-to-LWC mapping

**Requires:** Stream A (Data Model) must be deployed first. All custom objects, fields, CMTs, and platform events must exist.

**Specification:**

**`EncryptionService.cls`**
- Two public static methods: `String encrypt(String plaintext)` and `String decrypt(String ciphertext)`.
- Uses `Crypto.encryptWithManagedIV('AES256', key, Blob.valueOf(plaintext))` and the inverse.
- Key loaded from `tgint__Encryption_Key__mdt` where `Is_Active__c = true`, ordered by `DeveloperName DESC` (newest first).
- Encrypt always uses the first (newest) active key. Decrypt tries all active keys (supports key rotation).
- Both methods throw `EncryptionService.EncryptionException` on failure.
- Internal `@TestVisible` variable for key override in tests.

**`ChannelSetupController.cls`**
- `with sharing` context for all methods (admin-only operations, but respect org sharing).
- 11 `@AuraEnabled(cacheable=...)` methods:

1. `@AuraEnabled(cacheable=true) static List<ChannelTypeWrapper> listChannelTypes()` -- Query `tgint__Channel_Type__mdt`, return list. `ChannelTypeWrapper` inner class: `{ String apiName, String label, String messenger, String protocol, String configObjectName, Boolean supportsOutbound }`.

2. `@AuraEnabled static List<ChannelListWrapper> listChannels(String searchText, String channelType, String status)` -- Dynamic SOQL on `Messenger_Channel__c` with optional WHERE clauses. Returns list with fields: id, name (Display_Name__c), subLabel (External_Account_ID__c), channelType (Channel_Type_API_Name__c), status (Status__c), sessionStatus (Session_Status__c), isActive (Active__c), lastActivity (Last_Connected_At__c), errorNote (Session_Error_Detail__c), hasChatChildren (Boolean from subquery).

3. `@AuraEnabled static ChannelDetailWrapper getExistingChannel(String channelId)` -- Load channel + its config child. Used for wizard edit/resume. Returns all field values including decrypted tokens (for pre-populating masked inputs -- the LWC should mask them; the decrypted values are needed for the test connection step).

4. `@AuraEnabled static String createApp(String channelType, Map<String, String> payload)` -- Upsert the appropriate `*_App__c` based on channelType. Encrypt sensitive fields (API_Hash__c, App_Secret__c, Auth_Token__c). Return the App record ID. channelType values: `Telegram_Bot` -> `Telegram_App__c`, `WhatsApp_Cloud` / `Facebook_Messenger` / `Instagram_Direct` -> `WhatsApp_App__c`, `Twilio_SMS` -> `Twilio_App__c`, `Viber_Bot` -> no app object (credentials are on the config).

5. `@AuraEnabled static String createChannel(String appId, String channelType, Map<String, String> payload)` -- Insert or upsert `Messenger_Channel__c` with `Status__c = 'draft'`. Then insert or upsert the matching `*_Config__c` with encrypted token. payload keys: channelName, handle, token, assigneeId, assigneeType, autoLinkStrategy. Return Channel ID.

6. `@AuraEnabled static ConnectionTestResult testConnection(String channelId)` -- Load channel + config, decrypt token, make HTTP callout to provider's "get me" endpoint. Return `ConnectionTestResult` inner class: `{ Boolean success, String botId, String botUsername, String details, Integer roundTripMs }`.
   - Telegram: `GET https://api.telegram.org/bot{token}/getMe`
   - WhatsApp: `GET https://graph.facebook.com/v21.0/{phone_number_id}?access_token={token}`
   - Viber: `POST https://chatapi.viber.com/pa/get_account_info` with `X-Viber-Auth-Token` header
   - Twilio: `GET https://api.twilio.com/2010-04-01/Accounts/{account_sid}.json` with Basic auth
   - Messenger: `GET https://graph.facebook.com/v21.0/{page_id}?access_token={token}`
   - Instagram: `GET https://graph.facebook.com/v21.0/{ig_business_id}?access_token={token}`

7. `@AuraEnabled static void activateChannel(String channelId)` -- Update `Status__c = 'active'`, `Active__c = true`, `Session_Status__c = 'online'`. Insert audit log with `Event_Type__c = 'channel_activated'`.

8. `@AuraEnabled static WebhookRegistrationResult registerWebhook(String channelId)` -- Generate a webhook secret (UUID), store encrypted in config. Make HTTP callout to register webhook URL with provider.
   - Webhook URL pattern: `https://{site_base}/services/apexrest/tgint/webhook/{channelType}/{channelId}`
   - Telegram: `POST https://api.telegram.org/bot{token}/setWebhook` with `url`, `secret_token`, `allowed_updates`
   - WhatsApp/Messenger/Instagram: `POST https://graph.facebook.com/v21.0/{app_id}/subscriptions` with `callback_url`, `verify_token`, `fields`
   - Viber: `POST https://chatapi.viber.com/pa/set_webhook` with `url`, `auth_token`
   - Twilio: Update the Phone Number webhook URL via `POST https://api.twilio.com/2010-04-01/Accounts/{sid}/IncomingPhoneNumbers/{phone_sid}.json`
   - Return `WebhookRegistrationResult`: `{ Boolean success, String webhookUrl, String errorMessage }`. Insert audit log with `Event_Type__c = 'webhook_registered'`.

9. `@AuraEnabled static void toggleActive(String channelId, Boolean isActive)` -- Update `Active__c`.

10. `@AuraEnabled static void softDeleteChannel(String channelId)` -- Set `Active__c = false`, `Status__c = 'suspended'`. Attempt to unregister webhook with provider. Insert audit log.

11. `@AuraEnabled static void hardDeleteChannel(String channelId)` -- Verify zero `Messenger_Chat__c` children. Delete config record, then channel. Insert audit log.

12. `@AuraEnabled(cacheable=true) static List<AuditLogWrapper> getRecentAuditLogs(String channelId, Integer limitCount)` -- Query `Channel_Audit_Log__c` ordered by `Event_Timestamp__c DESC`. `channelId` nullable (all channels). Default limit 20.

**Per-provider webhook service classes:**
- `TelegramWebhookService.cls`, `WhatsAppWebhookService.cls`, `ViberWebhookService.cls`, `TwilioWebhookService.cls`, `MessengerWebhookService.cls` (Facebook), `InstagramWebhookService.cls`
- Each implements an interface `IWebhookRegistrationService` with method `WebhookRegistrationResult register(String channelId, String webhookUrl, String secret)` and `void unregister(String channelId)`.

**LWC Components:**

1. **`channelSetupWizard`** -- Five-step wizard.
   - `@api channelId` -- if provided, loads existing channel for edit/resume.
   - Uses `lightning-progress-indicator` with `type="path"` for the step bar.
   - Step 1: `channelTypeSelector` child -- grid of cards for the 6 Phase 1 channel types + 3 disabled Phase 2 types. Fires `typeselect` event.
   - Step 2: `channelAppForm` child -- fields vary by channelType. Telegram: App Name, API ID, API Hash, Owner Email. WhatsApp/Messenger/Instagram: App Name, Meta App ID, App Secret, Owner Email. Twilio: App Name, Account SID, Auth Token, Owner Email. Viber: skip step 2 (no app object). All sensitive fields use `type="password"`.
   - Step 3: `channelConfigForm` child -- Channel Name, Handle, Token (type="password"), Default Assignee (lightning-record-picker for User), Auto-link strategy (combobox: Lead by phone, Contact, Always orphan). Varies slightly by channel type.
   - Step 4: `channelConnectionTest` child -- "Run Test" button, shows success/failure card.
   - Step 5: `channelActivationForm` child -- Webhook URL (read-only), HMAC Secret (read-only), sharing group selector. "Activate" button triggers `activateChannel` + `registerWebhook`.
   - Footer: Back, Cancel, Next (becomes "Activate" on step 5).
   - On successful activation, fires `channelsaved` event with `{ channelId, status: 'active' }`.

2. **`messengerChannelList`** -- Full channels management view.
   - Search input (`lightning-input`), channel type filter (`lightning-combobox`), status filter (`lightning-combobox`), "New Channel" button.
   - `lightning-datatable` with columns: Name, Type, Status (rendered as badge), Active (checkbox), Messages (24h), Last Activity, Actions (row-level buttons).
   - Action buttons change by status: online -> Open, auth_required -> Reauth, error -> Logs, draft -> Resume Setup. Plus Edit and Delete on all.
   - Delete uses `lightning-modal` for confirmation with hard-delete checkbox option.
   - Below the table: `channelAuditLog` child component.
   - When "New Channel" or "Resume Setup" or "Edit" is clicked, renders `channelSetupWizard` in place of the list.

3. **`channelAuditLog`** -- Simple `lightning-datatable` showing recent audit events. `@api channelId` for filtering.

**Deliverables:**
```
force-app/main/default/classes/EncryptionService.cls
force-app/main/default/classes/EncryptionService.cls-meta.xml
force-app/main/default/classes/ChannelSetupController.cls
force-app/main/default/classes/ChannelSetupController.cls-meta.xml
force-app/main/default/classes/TelegramWebhookService.cls (+ meta)
force-app/main/default/classes/WhatsAppWebhookService.cls (+ meta)
force-app/main/default/classes/ViberWebhookService.cls (+ meta)
force-app/main/default/classes/TwilioWebhookService.cls (+ meta)
force-app/main/default/classes/MessengerWebhookService.cls (+ meta)
force-app/main/default/classes/InstagramWebhookService.cls (+ meta)
force-app/main/default/classes/IWebhookRegistrationService.cls (+ meta)
force-app/main/default/lwc/channelSetupWizard/
force-app/main/default/lwc/channelTypeSelector/
force-app/main/default/lwc/channelAppForm/
force-app/main/default/lwc/channelConfigForm/
force-app/main/default/lwc/channelConnectionTest/
force-app/main/default/lwc/channelActivationForm/
force-app/main/default/lwc/messengerChannelList/
force-app/main/default/lwc/channelAuditLog/
```

**Acceptance criteria:**
1. `sf project deploy start --source-dir MessageForge.SalesForce/force-app --dry-run` passes.
2. All Apex classes compile without errors.
3. All LWCs compile without errors (check via `sf lightning lint force-app/main/default/lwc/` or dry-run deploy).
4. EncryptionService can encrypt and decrypt a test string.
5. Wizard steps are navigable and form inputs use `lightning-input` / `lightning-combobox` base components.

**Constraints:**
- Namespace prefix: `tgint__` for all custom objects/fields/events
- API version: `62.0`
- Package type: 2GP managed package
- Format: SFDX source format
- Target directory: `MessageForge.SalesForce/force-app/main/default/`
- No external infrastructure -- everything runs inside Salesforce
- AES-256 encryption for all stored credentials
- All LWC must use `lightning-*` base components and SLDS, not custom CSS
- All Apex class meta.xml files must have `<apiVersion>62.0</apiVersion>` and `<status>Active</status>`
- All HTTP callouts must be wrapped in try-catch with meaningful error messages

---

### Stream C -- Inbound Handlers

You are a Salesforce Apex developer working on the MessageForge managed package. Your task is to build the inbound webhook handler that receives messages from 6 messenger platforms, validates HMAC signatures, parses provider-specific payloads into a normalized format, persists messages, downloads media, and publishes platform events.

**Goal:** Deliver `MessengerInboundAPI.cls` (@RestResource), `HMACValidator.cls`, 6 per-channel parser classes, normalized DTO classes, media download service, and the `InboundMessageTrigger`.

**Files to read first:**
- `MessageForge.SalesForce/CLAUDE.md` -- subsystem conventions
- `MessageForge.Documentation/handoff/messageforge-phase1-technical-handoff.md` -- sections 4, 7, 8 (inbound flow, media handling, security)
- `MessageForge.Documentation/handoff/sf-data-model.md` -- Message, Chat, Attachment object fields

**Requires:** Stream A (Data Model) must be deployed. Stream B (EncryptionService) must be deployed for token decryption. If Stream B is not yet complete, create a local stub of `EncryptionService.cls` with the same method signatures that returns the input unchanged (for compilation).

**Specification:**

**`MessengerInboundAPI.cls`**
- `@RestResource(urlMapping='/tgint/webhook/*')`
- `global with sharing class MessengerInboundAPI`
- `@HttpPost global static void doPost()`:
  1. Extract `channelType` and `channelId` from `RestContext.request.requestURI`. URI format: `/services/apexrest/tgint/webhook/{channelType}/{channelId}`. Parse by splitting on `/`.
  2. SOQL: `SELECT Id, Active__c, Is_Compromised__c, Channel_Type_API_Name__c FROM tgint__Messenger_Channel__c WHERE Id = :channelId LIMIT 1`. If not found, return 404. If `Active__c = false` or `Is_Compromised__c = true`, return 410.
  3. Call `HMACValidator.verify(RestContext.request.requestBody, RestContext.request.headers, channel)`. If verification fails, insert `Channel_Audit_Log__c` with `Event_Type__c = 'hmac_verification_failed'` and return 401.
  4. Dispatch to the appropriate parser based on `channelType`: `Telegram_Bot` -> `TelegramInboundParser`, `WhatsApp_Cloud` -> `WhatsAppInboundParser`, `Viber_Bot` -> `ViberInboundParser`, `Twilio_SMS` -> `TwilioInboundParser`, `Facebook_Messenger` -> `MessengerInboundParser`, `Instagram_Direct` -> `InstagramInboundParser`.
  5. Parser returns `List<NormalizedInboundDTO>` (a single webhook payload can contain multiple messages, e.g., WhatsApp batches).
  6. For each DTO: upsert `Messenger_Chat__c` by `Chat_External_ID__c`. If new chat, run Lead/Contact matching: normalize sender phone to E.164, SOQL Contact by Phone/MobilePhone, then Lead by Phone/MobilePhone. First match wins, prefer Contact.
  7. Insert `Messenger_Message__c`. If DTO has attachments, insert `Messenger_Attachment__c` records. If media files need downloading (URL provided by provider), enqueue `MediaDownloadQueueable`.
  8. Update `Messenger_Chat__c`: `Last_Message_At__c = NOW()`, `Last_Message_Preview__c = first 255 chars of message text`, `Unread_Count__c += 1`.
  9. Publish `tgint__Inbound_Message__e` with: `Text__c`, `Chat_SF_ID__c`, `Chat_External_ID__c`, `Sender_Name__c`, `Sender_External_ID__c`, `Message_Type__c`, `Media_URL__c`, `Media_MIME_Type__c`, `Protocol__c`, `Sent_At__c`, `Message_SF_ID__c`.
  10. Return 200 with empty body.

- `@HttpGet global static void doGet()`:
  1. Used for WhatsApp/Messenger/Instagram webhook verification.
  2. Read `hub.mode`, `hub.verify_token`, `hub.challenge` from `RestContext.request.params`.
  3. If `hub.mode = 'subscribe'`: load channel by ID from URI, load config, compare `hub.verify_token` with stored `Verify_Token__c`. If match, return `hub.challenge` with 200. Else return 403.
  4. If no hub params, return 400.

**`HMACValidator.cls`**
- `public with sharing class HMACValidator`
- `public static void verify(Blob requestBody, Map<String, String> headers, tgint__Messenger_Channel__c channel)` -- throws `HMACValidator.HMACException` on failure.
- Per-channel strategies:
  - **Telegram_Bot:** Header `X-Telegram-Bot-Api-Secret-Token`. Compare with `Telegram_Bot_Config__c.Webhook_Secret__c` (decrypted). Simple string equality.
  - **WhatsApp_Cloud / Facebook_Messenger / Instagram_Direct:** Header `X-Hub-Signature-256` (format: `sha256=<hex>`). Compute `HMAC-SHA256(body, App_Secret)` from `WhatsApp_App__c.App_Secret__c` (decrypted). Compare hex digests.
  - **Viber_Bot:** Header `X-Viber-Content-Signature`. Compute `HMAC-SHA256(body, Auth_Token)` from `Viber_Config__c.Auth_Token__c` (decrypted). Compare hex digests.
  - **Twilio_SMS:** Header `X-Twilio-Signature`. Compute HMAC-SHA1 of the full request URL + sorted POST params using `Twilio_App__c.Auth_Token__c` (decrypted). Compare Base64 outputs.
- Use `Crypto.generateMac('HmacSHA256', ...)` or `Crypto.generateMac('HmacSHA1', ...)` as appropriate.
- Load config records via SOQL within the verify method (add 1-2 SOQL to the budget).

**Parser Interface and Classes:**
- `public interface IInboundParser { List<NormalizedInboundDTO> parse(Blob requestBody, Map<String, String> headers); }`
- 6 implementations, each parsing the provider's JSON payload:

  - **`TelegramInboundParser`**: JSON has `message.text`, `message.from.id`, `message.from.first_name`, `message.chat.id`, `message.message_id`. Media: `message.photo` (array, take last = highest resolution), `message.document`, `message.video`, `message.voice`, `message.audio`, `message.sticker`. Map to `NormalizedInboundDTO`.

  - **`WhatsAppInboundParser`**: JSON has `entry[].changes[].value.messages[]`. Each message has: `id`, `from` (phone), `timestamp`, `type` (text/image/video/document/audio/sticker/location/contacts), `text.body` or media object with `id`. Status updates in `entry[].changes[].value.statuses[]` -- these update existing message `Delivery_Status__c` rather than creating new messages.

  - **`ViberInboundParser`**: JSON has `event` (message/seen/delivered), `sender.id`, `sender.name`, `message.text`, `message.type` (text/picture/video/file/sticker). Message token in `message_token`.

  - **`TwilioInboundParser`**: POST form body (not JSON). Params: `From`, `To`, `Body`, `MessageSid`, `NumMedia`, `MediaUrl0..N`, `MediaContentType0..N`.

  - **`MessengerInboundParser`**: JSON same structure as WhatsApp (Meta Graph API). `entry[].messaging[].message.text`, `entry[].messaging[].sender.id`, `entry[].messaging[].message.mid`. Attachments in `message.attachments[].type` and `message.attachments[].payload.url`.

  - **`InstagramInboundParser`**: JSON similar to Messenger. `entry[].messaging[].message.text`, `entry[].messaging[].sender.id`. Media in `message.attachments[]`.

**`NormalizedInboundDTO`**
- `public class NormalizedInboundDTO`:
  - `String chatExternalId` -- unique chat identifier from provider
  - `String chatTitle` -- display name
  - `String chatType` -- private, group, supergroup, channel
  - `String messageExternalId` -- unique message ID from provider
  - `String senderName`
  - `String senderExternalId` -- phone or user ID
  - `String messageText`
  - `String messageType` -- text, photo, video, voice, document, sticker, album
  - `Datetime sentAt`
  - `String replyToExternalId` -- nullable
  - `Boolean isEdited`
  - `List<NormalizedAttachmentDTO> attachments`

**`NormalizedAttachmentDTO`**
- `public class NormalizedAttachmentDTO`:
  - `String mediaExternalId` -- provider file ID
  - `String downloadUrl` -- direct URL or file ID to resolve
  - `String mimeType`
  - `String fileName`
  - `Long fileSize` -- bytes, nullable
  - `Integer sortOrder`
  - `Integer durationSeconds` -- for voice/video
  - `Integer width`
  - `Integer height`

**`MediaDownloadService.cls`**
- `public class MediaDownloadService`:
  - `public static void downloadAndAttach(String messageId, List<NormalizedAttachmentDTO> attachments, String channelType, String configId)` -- for each attachment: resolve download URL (Telegram requires a `getFile` API call first), HTTP GET the file, check size (<= 5MB for sync, 5-12MB enqueue Queueable, >12MB set `Last_Error__c = 'media_too_large_phase2'`). Create `ContentVersion`, link via `ContentDocumentLink` to the `Messenger_Message__c`, update `Messenger_Attachment__c.Media_URL__c` with `/sfc/servlet.shepherd/version/download/{contentVersionId}`.

**`MediaDownloadQueueable.cls`**
- `public class MediaDownloadQueueable implements Queueable, Database.AllowsCallouts` -- handles 5-12MB files that exceed sync heap. Same logic as `MediaDownloadService` but in async context.

**`InboundMessageTrigger.trigger`**
- On `Messenger_Message__c` after insert.
- For `Direction__c = 'inbound'` messages: send `Messaging.CustomNotification` to `Assigned_User__c` of the parent `Messenger_Chat__c` (if populated).
- Trigger handler class: `InboundMessageTriggerHandler.cls`.

**Deliverables:**
```
force-app/main/default/classes/MessengerInboundAPI.cls (+ meta)
force-app/main/default/classes/HMACValidator.cls (+ meta)
force-app/main/default/classes/IInboundParser.cls (+ meta)
force-app/main/default/classes/TelegramInboundParser.cls (+ meta)
force-app/main/default/classes/WhatsAppInboundParser.cls (+ meta)
force-app/main/default/classes/ViberInboundParser.cls (+ meta)
force-app/main/default/classes/TwilioInboundParser.cls (+ meta)
force-app/main/default/classes/MessengerInboundParser.cls (+ meta)
force-app/main/default/classes/InstagramInboundParser.cls (+ meta)
force-app/main/default/classes/NormalizedInboundDTO.cls (+ meta)
force-app/main/default/classes/NormalizedAttachmentDTO.cls (+ meta)
force-app/main/default/classes/MediaDownloadService.cls (+ meta)
force-app/main/default/classes/MediaDownloadQueueable.cls (+ meta)
force-app/main/default/classes/InboundMessageTriggerHandler.cls (+ meta)
force-app/main/default/triggers/InboundMessageTrigger.trigger (+ meta)
```

**Acceptance criteria:**
1. `sf project deploy start --source-dir MessageForge.SalesForce/force-app --dry-run` passes.
2. All Apex classes compile without errors.
3. `MessengerInboundAPI` correctly routes based on URI pattern.
4. HMAC validation logic matches each provider's documented algorithm.
5. Each parser correctly maps provider JSON to `NormalizedInboundDTO`.

**Constraints:**
- Namespace prefix: `tgint__` for all custom objects/fields/events
- API version: `62.0`
- Package type: 2GP managed package
- Format: SFDX source format
- Target directory: `MessageForge.SalesForce/force-app/main/default/`
- No external infrastructure
- The REST endpoint runs in Guest User context (`without sharing` for DML operations on objects, but `HMACValidator` runs before any DML)
- Total webhook handler execution time target: <2 seconds, hard ceiling 5 seconds
- Maximum 5 SOQL queries per webhook invocation
- Maximum 4 DML statements per webhook invocation

---

### Stream D -- Outbound Flow

You are a Salesforce Apex developer working on the MessageForge managed package. Your task is to build the outbound message sending pipeline: a service class that dispatches messages to per-channel sender classes, a Queueable for async execution with retry, delivery status tracking via platform events, and Named Credential metadata.

**Goal:** Deliver `MessengerOutboundService.cls`, `MessengerOutboundQueueable.cls`, 6 per-channel sender classes, `DeliveryStatusTrigger`, and Named Credential metadata templates.

**Files to read first:**
- `MessageForge.SalesForce/CLAUDE.md`
- `MessageForge.Documentation/handoff/messageforge-phase1-technical-handoff.md` -- section 5 (outbound flow)
- `MessageForge.Documentation/handoff/sf-data-model.md`

**Requires:** Stream A (Data Model). Stream B (EncryptionService for token decryption).

**Specification:**

**`MessengerOutboundService.cls`**
- `public with sharing class MessengerOutboundService`
- `public static void send(Id messageId)` -- loads `Messenger_Message__c` + parent `Messenger_Chat__c` + `Messenger_Channel__c` + config, dispatches to the appropriate sender.
- Uses a factory pattern: `getHandler(String channelType)` returns an `IOutboundSender` implementation.
- After successful send: update `Messenger_Message__c.Delivery_Status__c = 'sent'`, `Sent_At__c = Datetime.now()`, `Message_External_ID__c = provider's message ID`.
- After failure (4xx): update `Delivery_Status__c = 'failed'`, `Last_Error__c = parsed error message`.
- After transient failure (5xx): throw exception so the Queueable can retry.
- After any status change: publish `tgint__Message_Delivery_Status__e` with `Message_SF_ID__c = messageId`, `Message_External_ID__c`, `Status__c` (SENT / FAILED / RETRYING), `Error_Code__c`, `Error_Detail__c`, `Timestamp__c`.

**`MessengerOutboundQueueable.cls`**
- `public class MessengerOutboundQueueable implements Queueable, Database.AllowsCallouts`
- Constructor: `MessengerOutboundQueueable(Id messageId, Integer attemptNumber)`
- `execute(QueueableContext context)`:
  1. Call `MessengerOutboundService.send(messageId)`.
  2. On success: done.
  3. On transient failure: if `attemptNumber < 3`, enqueue a new `MessengerOutboundQueueable(messageId, attemptNumber + 1)`. Publish PE with `Status__c = 'RETRYING'`.
  4. On permanent failure: update message to `failed`, publish PE. Do not retry.
  5. Check `Messenger_Channel__c.Is_Compromised__c` before sending -- if true, abort immediately and set `Delivery_Status__c = 'failed'`, `Last_Error__c = 'channel_compromised'`.

**`IOutboundSender` interface:**
- `public interface IOutboundSender { OutboundResult send(tgint__Messenger_Message__c message, tgint__Messenger_Chat__c chat, tgint__Messenger_Channel__c channel, SObject config); }`
- `OutboundResult` inner class: `{ Boolean success, Boolean isTransient, String externalMessageId, String errorCode, String errorDetail, Integer httpStatusCode }`

**6 sender implementations:**

1. **`TelegramSender.cls`**: `POST https://api.telegram.org/bot{token}/sendMessage` with JSON body `{ chat_id, text, reply_to_message_id? }`. For media: `sendPhoto`, `sendDocument`, `sendVideo`, `sendVoice` with `multipart/form-data` or by URL. Token from decrypted `Telegram_Bot_Config__c.Bot_Token__c`.

2. **`WhatsAppSender.cls`**: `POST https://graph.facebook.com/v21.0/{phone_number_id}/messages` with JSON body `{ messaging_product: "whatsapp", to: "{recipient_phone}", type: "text", text: { body: "{text}" } }`. For templates: `{ type: "template", template: { name, language: { code }, components: [{ type: "body", parameters: [...] }] } }`. Bearer token from `WhatsApp_Config__c.Access_Token__c`.

3. **`ViberSender.cls`**: `POST https://chatapi.viber.com/pa/send_message` with JSON body `{ receiver: "{user_id}", type: "text", text: "{text}" }`. Auth header `X-Viber-Auth-Token` from `Viber_Config__c.Auth_Token__c`.

4. **`TwilioSender.cls`**: `POST https://api.twilio.com/2010-04-01/Accounts/{sid}/Messages.json` with form body `From={phone}&To={recipient}&Body={text}`. Basic auth using Account SID + Auth Token from `Twilio_App__c`.

5. **`MessengerSender.cls`** (Facebook): `POST https://graph.facebook.com/v21.0/me/messages` with JSON body `{ recipient: { id: "{psid}" }, message: { text: "{text}" } }`. Page Access Token from `Messenger_Config__c.Page_Access_Token__c`.

6. **`InstagramSender.cls`**: `POST https://graph.facebook.com/v21.0/me/messages` with same structure as Messenger. Page Access Token from `Instagram_Config__c.Page_Access_Token__c`.

**`DeliveryStatusTrigger.trigger`**
- On `Messenger_Message__c` after update.
- When `Delivery_Status__c` changes (compare `Trigger.oldMap` vs `Trigger.newMap`): publish `tgint__Message_Delivery_Status__e` with the new status.
- Trigger handler class: `DeliveryStatusTriggerHandler.cls`.
- Note: The Queueable also publishes the PE directly after its send attempt. The trigger serves as a safety net for status changes made by other mechanisms (e.g., inbound webhook status updates from WhatsApp).

**Named Credential Metadata:**
- Create External Credential and Named Credential metadata files for each provider. These are templates -- the actual per-channel credentials are created at runtime by the wizard. The metadata files define the authentication protocols:
  - Telegram: Custom auth (token in URL path)
  - WhatsApp/Messenger/Instagram: Bearer token (OAuth 2.0 Custom)
  - Viber: Custom header auth
  - Twilio: Basic auth (username + password)

**Deliverables:**
```
force-app/main/default/classes/MessengerOutboundService.cls (+ meta)
force-app/main/default/classes/MessengerOutboundQueueable.cls (+ meta)
force-app/main/default/classes/IOutboundSender.cls (+ meta)
force-app/main/default/classes/OutboundResult.cls (+ meta)
force-app/main/default/classes/TelegramSender.cls (+ meta)
force-app/main/default/classes/WhatsAppSender.cls (+ meta)
force-app/main/default/classes/ViberSender.cls (+ meta)
force-app/main/default/classes/TwilioSender.cls (+ meta)
force-app/main/default/classes/MessengerSender.cls (+ meta)
force-app/main/default/classes/InstagramSender.cls (+ meta)
force-app/main/default/classes/DeliveryStatusTriggerHandler.cls (+ meta)
force-app/main/default/triggers/DeliveryStatusTrigger.trigger (+ meta)
force-app/main/default/namedCredentials/ (template files)
force-app/main/default/externalCredentials/ (template files)
```

**Acceptance criteria:**
1. `sf project deploy start --source-dir MessageForge.SalesForce/force-app --dry-run` passes.
2. All Apex classes compile without errors.
3. `MessengerOutboundQueueable` respects the 3-retry limit and the `Is_Compromised__c` kill switch.
4. Each sender class constructs the correct HTTP request structure for its provider's API.
5. `DeliveryStatusTrigger` correctly publishes PE on status change.

**Constraints:**
- Namespace prefix: `tgint__` for all custom objects/fields/events
- API version: `62.0`
- Package type: 2GP managed package
- Format: SFDX source format
- Target directory: `MessageForge.SalesForce/force-app/main/default/`
- No external infrastructure
- All HTTP callouts must be inside `Database.AllowsCallouts` context
- Maximum 3 retry attempts per outbound message
- Kill switch (`Is_Compromised__c`) check before every send attempt

---

### Stream E -- LWC Chat UI

You are a Salesforce LWC developer working on the MessageForge managed package. Your task is to build the complete chat user interface: the main conversations view, the embedded record-page variant, all child components (bubbles, filter, composer, template picker, media viewer), the utility bar inbox popup, and the Apex controller backend.

**Goal:** Deliver `messengerChat` LWC (full + embedded modes), `messengerChatBubble`, `messengerChannelFilter`, `messengerComposer`, `messengerTemplatePicker`, `messengerMediaViewer`, `messengerInbox`, `messengerRecordLinker`, `messengerChatList`, `messengerChatThread`, and `MessengerController.cls`.

**Files to read first:**
- `MessageForge.SalesForce/CLAUDE.md`
- `MessageForge.Documentation/handoff/messageforge-phase1-technical-handoff.md` -- sections 5, 6, 9, 10
- `MessageForge.Documentation/handoff/PROTOTYPE-HANDOFF.md` -- full UI mapping reference
- `MessageForge.Documentation/handoff/sf-data-model.md`

**Requires:** Stream A (Data Model). Streams C and D are needed for end-to-end message flow but the LWCs can be built and deployed independently -- they call Apex methods that return data from the objects created in Stream A. If Streams C/D are not yet deployed, the LWCs will still compile and render; they just will not have real message data.

**Specification:**

**`MessengerController.cls`**
- `public with sharing class MessengerController`
- All methods enforce sharing rules -- users only see chats they have access to via `Channel_User_Access__c` + Apex Managed Sharing.

Methods (all `@AuraEnabled`):

1. `getChatList(String filtersJson, Integer pageSize, String lastMessageCursor)`:
   - `filtersJson` is a JSON string: `{ "channelIds": ["id1"], "channelType": "Telegram_Bot", "status": "open", "searchText": "maria", "unreadOnly": true }`. All fields nullable.
   - `pageSize` default 20, max 50.
   - `lastMessageCursor` is an ISO datetime string for keyset pagination (null for first page).
   - Returns JSON-serializable wrapper: `ChatListResult { List<ChatWrapper> chats, Boolean hasMore }`.
   - `ChatWrapper`: `{ String id, String chatTitle, String channelId, String channelType, String channelName, String leadId, String leadName, String contactId, String contactName, String assignedUserId, String assignedUserName, Datetime lastMessageAt, String lastMessagePreview, Integer unreadCount, String status, String chatExternalId }`.
   - SOQL: Query `Messenger_Chat__c` with dynamic WHERE clause from filters. `ORDER BY Last_Message_At__c DESC LIMIT :pageSize+1`. Query `pageSize + 1` records to determine `hasMore` (if result count > pageSize, hasMore = true, return only first pageSize).

2. `getChatsForRecord(String recordId)`:
   - Query `Messenger_Chat__c WHERE Lead__c = :recordId OR Contact__c = :recordId ORDER BY Last_Message_At__c DESC`.
   - Returns `List<ChatWrapper>`.

3. `getRecentMessages(String chatId, String sinceCursor, Integer pageSize)`:
   - `sinceCursor` is ISO datetime string (null for latest page).
   - `pageSize` default 50.
   - Returns `MessageListResult { List<MessageWrapper> messages, Boolean hasMore }`.
   - `MessageWrapper`: `{ String id, String text, String direction, String messageType, String senderName, String senderExternalId, String deliveryStatus, String deliveryError, String lastError, Datetime sentAt, Boolean isEdited, String replyToExternalId, String mediaUrl, String mediaMimeType, String templateId, String templateName, List<AttachmentWrapper> attachments }`.
   - `AttachmentWrapper`: `{ String id, String mediaUrl, String mediaMimeType, String fileName, Decimal fileSize, Integer sortOrder, String thumbnailUrl, Integer durationSeconds, Integer width, Integer height }`.
   - SOQL: Query `Messenger_Message__c` with subquery on `Messenger_Attachment__c`. Order DESC by `Sent_At__c`. Reverse the list in Apex before returning so messages are in chronological order.

4. `sendMessage(String chatId, String text, List<String> attachmentIds)`:
   - Validate: user has access to the chat's channel (query `Channel_User_Access__c`). Chat `Status__c` must not be 'archived'.
   - Insert `Messenger_Message__c` with `Direction__c = 'outbound'`, `Delivery_Status__c = 'pending'`, `Message_Type__c = 'text'` (or 'photo'/'document' if attachments provided), `Sent_At__c = Datetime.now()`.
   - If `attachmentIds` provided: create `Messenger_Attachment__c` records linking to the ContentVersion IDs.
   - Update `Messenger_Chat__c.Last_Message_At__c` and `Last_Message_Preview__c`.
   - Enqueue `MessengerOutboundQueueable(messageId, 1)`.
   - Return the created `MessageWrapper`.

5. `sendTemplate(String chatId, String templateId, String variablesJson)`:
   - `variablesJson` is `{"1": "John", "2": "tomorrow"}`.
   - Load template, validate `Approval_Status__c = 'approved'` and `Is_Active__c = true`.
   - Insert `Messenger_Message__c` with `Message_Type__c = 'template'`, `Template__c = templateId`. Render the body with variables substituted and store in `Message_Text__c`.
   - Enqueue `MessengerOutboundQueueable`.
   - Return `MessageWrapper`.

6. `markAsRead(String chatId)`:
   - Update `Messenger_Chat__c.Unread_Count__c = 0`.

7. `linkToRecord(String chatId, String recordId, String recordType)`:
   - If `recordType = 'Lead'`: update `Messenger_Chat__c.Lead__c = recordId`, `Contact__c = null`.
   - If `recordType = 'Contact'`: update `Contact__c = recordId`, `Lead__c = null`.
   - Insert audit log: `Event_Type__c = 'manual_link_changed'`.

8. `getTemplates(String channelId)`:
   - Query `Messenger_Template__c WHERE (Channel__c = :channelId OR Channel__c = null) AND Approval_Status__c = 'approved' AND Is_Active__c = true ORDER BY Template_Name__c`.
   - Returns `List<TemplateWrapper>`: `{ String id, String name, String language, String body, String header, String footer, String category, String variableSchema, String approvalStatus }`.

9. `getInboxChats(String tabFilter)`:
   - `tabFilter`: 'new' (Unread_Count__c > 0), 'live' (TODO: last msg outbound), 'pending' (TODO: last msg inbound).
   - For 'live' and 'pending', use a subquery or formula approach. Simplest: query `Messenger_Chat__c` and check `Last_Message_Preview__c` direction indicator (add a `Last_Message_Direction__c` formula field, or query the last message in Apex).
   - Returns `List<InboxChatWrapper>`.

10. `sendQuickReply(String chatId, String text)`:
    - Delegates to `sendMessage(chatId, text, null)`.

**LWC Components:**

1. **`messengerChat`** (container -- main component):
   - `@api recordId` -- when provided, component is in embedded mode (filtered to chats for this Lead/Contact).
   - `@api mode` -- 'full' (default) or 'embedded'.
   - In full mode: renders `messengerChannelFilter` + `messengerChatList` (left pane) + `messengerChatThread` + `messengerComposer` (right pane). Two-column flex layout.
   - In embedded mode: renders `lightning-tabset` with one tab per chat linked to the record. Each tab contains `messengerChatThread` + `messengerComposer`. No left pane.
   - Subscribes to `/event/tgint__Inbound_Message__e` and `/event/tgint__Message_Delivery_Status__e` via `lightning/empApi` in `connectedCallback()`. Unsubscribes in `disconnectedCallback()`.
   - On `Inbound_Message__e`: if `Chat_SF_ID__c` matches active chat, append message to thread. Otherwise, update chat list (increment unread, move chat to top).
   - On `Message_Delivery_Status__e`: if `Message_SF_ID__c` matches a message in the active thread, update its `deliveryStatus` property reactively.
   - Handles `chatselect` event from `messengerChatList`: loads messages for selected chat via `getRecentMessages`.
   - Handles `messagesent` event from `messengerComposer`: calls `sendMessage`, appends result to thread with pending status.
   - Handles `mediaclick` event from `messengerChatBubble`: opens `messengerMediaViewer`.
   - Handles `linkrecord` from `messengerRecordLinker`.
   - Chat header shows: avatar, name, channel badge, online status, linked record pill (with "Link" or "Re-link" action).
   - Pagination: initial load gets 50 messages. "Load older messages" link at top of thread calls `getRecentMessages` with cursor.

2. **`messengerChatBubble`**:
   - `@api message` -- object with all message fields.
   - Renders differently by `message.direction`:
     - `inbound`: left-aligned, light background.
     - `outbound`: right-aligned, blue background.
   - Renders differently by `message.messageType`:
     - `text`: plain text with newlines preserved.
     - `photo`/`image`: `<img>` tag with `src` = `message.mediaUrl` or first attachment's `mediaUrl`, `loading="lazy"`, `max-width: 240px`, `max-height: 200px`. Wrapped in clickable container that fires `mediaclick`.
     - `video`: `<video>` tag with poster, play button overlay. Click fires `mediaclick`.
     - `voice`/`audio`: `<audio>` tag with controls. Duration display.
     - `document`/`file`: file card with icon, name, size. Download link.
     - `sticker`: small image (120x120).
     - `template`: rendered body with header/footer.
     - `album`: grid of thumbnail images.
     - `system`: centered, muted text, no bubble.
   - Bottom-right of bubble: timestamp + delivery status icon for outbound:
     - `pending`: clock icon
     - `sent`: single checkmark
     - `delivered`: double checkmark (gray)
     - `read`: double checkmark (blue)
     - `failed`: red X icon with tooltip showing error
   - If `message.isEdited`: show "(edited)" label.
   - If `message.lastError = 'media_too_large_phase2'`: render a styled placeholder card: "Media file exceeds current size limit. This file will be available in a future update." with an icon.
   - If `message.replyToExternalId`: render a reply-to quote block above the message content.

3. **`messengerChannelFilter`**:
   - `@api channels` -- list of channel objects with `{ id, name, channelType }`.
   - `@api selectedType`, `@api selectedChannelId`.
   - Renders a row of chip buttons (All + one per channel type present in channels). Uses `lightning-badge` or `lightning-button` for chips.
   - Below chips: `lightning-combobox` with channels filtered by selected type.
   - Fires `filterchange` event on any selection change.

4. **`messengerChatList`**:
   - `@api chats` -- list of chat objects.
   - `@api activeChatId`.
   - Renders scrollable list of chat rows. Each row: avatar (initials, colored), name with channel icon, timestamp, message preview, unread badge.
   - Active chat highlighted.
   - "Load more" button at bottom if `hasMore`.
   - Fires `chatselect` event on row click.

5. **`messengerChatThread`**:
   - `@api messages` -- list of message objects.
   - `@api chatId`.
   - Renders `messengerChatBubble` for each message.
   - Auto-scrolls to bottom on new messages.
   - "Load older messages" link at top.
   - Fires `mediaclick` (bubbled from child bubbles) and `loadmore` events.

6. **`messengerComposer`**:
   - `@api chatId`, `@api channelType`, `@api channelId`, `@api disabled`.
   - `textarea` (`lightning-textarea` or native `<textarea>`) with Enter-to-send behavior (Shift+Enter for newline). Use `onkeydown` handler.
   - Attachment button: uses `lightning-file-upload` to upload to ContentVersion, stores IDs.
   - Template button: visible only when `channelType` is `WhatsApp_Cloud`. Fires `templaterequest` which opens `messengerTemplatePicker`.
   - Send button (disabled when textarea is empty and no attachments).
   - Fires `messagesent` event with `{ text, attachmentIds }`.

7. **`messengerTemplatePicker`**:
   - `@api channelId`.
   - Loads templates via `MessengerController.getTemplates(channelId)`.
   - Two-panel layout: left = template list (searchable), right = preview + variable inputs.
   - Each template shows name, language, category badge.
   - On selection: parse `variableSchema` JSON, render `lightning-input` for each variable placeholder.
   - Preview area shows template body with variables substituted in real time.
   - "Send Template" button fires `templateselect` with `{ templateId, variables }`.
   - "Cancel" button fires `templatecancel`.

8. **`messengerMediaViewer`**:
   - `@api contentVersionId`, `@api mimeType`, `@api fileName`.
   - Full-screen overlay (modal/lightbox).
   - For images: `<img>` with `src="/sfc/servlet.shepherd/version/download/{contentVersionId}"`.
   - For video: `<video>` with same URL pattern.
   - For audio: `<audio>` with controls.
   - For documents: download link.
   - Close button (X or click outside).
   - Fires `close` event.

9. **`messengerInbox`** (utility bar component):
   - Registered as a utility item in the App Builder.
   - Three tabs: New Chats, Live Chats, Pending.
   - Each tab calls `MessengerController.getInboxChats(tabFilter)`.
   - Rows show: avatar, name, channel, preview, unread badge, "Open Chat" button.
   - "Open Chat" uses `NavigationMixin` to navigate to the `Messenger_Chat__c` record.
   - Quick reply input at bottom: sends to the top chat in the current tab.
   - Subscribes to `Inbound_Message__e` to update badge count in real time.
   - Uses `@wire` for initial data, refreshes on PE receipt.

10. **`messengerRecordLinker`**:
    - `@api chatId`, `@api currentLeadId`, `@api currentContactId`.
    - Renders a `lightning-record-picker` that searches Lead and Contact.
    - On selection, calls `MessengerController.linkToRecord`.
    - Fires `linkrecord` event.

11. **`messengerChatThread`** (internal component for thread rendering).

**Deliverables:**
```
force-app/main/default/classes/MessengerController.cls (+ meta)
force-app/main/default/lwc/messengerChat/
force-app/main/default/lwc/messengerChatBubble/
force-app/main/default/lwc/messengerChannelFilter/
force-app/main/default/lwc/messengerChatList/
force-app/main/default/lwc/messengerChatThread/
force-app/main/default/lwc/messengerComposer/
force-app/main/default/lwc/messengerTemplatePicker/
force-app/main/default/lwc/messengerMediaViewer/
force-app/main/default/lwc/messengerInbox/
force-app/main/default/lwc/messengerRecordLinker/
```

Each LWC directory must contain at minimum: `componentName.html`, `componentName.js`, `componentName.js-meta.xml`. Add `componentName.css` only when SLDS utility classes are insufficient.

**Acceptance criteria:**
1. `sf project deploy start --source-dir MessageForge.SalesForce/force-app --dry-run` passes.
2. All LWCs compile without errors.
3. `MessengerController.cls` compiles without errors.
4. `messengerChat` renders in both full and embedded modes.
5. All child components fire and handle events correctly per the specification.
6. Platform Event subscriptions use `lightning/empApi` with proper subscribe/unsubscribe lifecycle.
7. All components use `lightning-*` base components (not custom HTML elements for inputs, buttons, etc.).

**Constraints:**
- Namespace prefix: `tgint__` for all custom objects/fields/events
- API version: `62.0`
- Package type: 2GP managed package
- Format: SFDX source format
- Target directory: `MessageForge.SalesForce/force-app/main/default/`
- No external infrastructure
- All LWC must use `lightning-*` base components and SLDS, not custom CSS (minimal CSS allowed only for layout that SLDS cannot express)
- Enter-to-send, Shift+Enter for newline in the composer
- Lazy image loading via `loading="lazy"` on `<img>` tags
- `messengerChat` js-meta.xml must expose it for: `lightning__RecordPage` (for embedded mode), `lightning__AppPage` (for full mode), and `lightning__UtilityBar` (for inbox)
- `messengerInbox` js-meta.xml must expose it for: `lightning__UtilityBar`

---

### Stream F -- Security & Sharing

You are a Salesforce Apex developer working on the MessageForge managed package. Your task is to build the Apex Managed Sharing system, the channel access trigger, the compromised channel kill switch trigger (Phase 1 stub), and the audit log utility service.

**Goal:** Deliver `ChannelAccessService.cls`, `ChannelAccessTrigger`, `ChannelCompromiseTrigger` (Phase 1 stub), `ChannelAuditService.cls`, and verify the sharing model works correctly.

**Files to read first:**
- `MessageForge.SalesForce/CLAUDE.md`
- `MessageForge.Documentation/handoff/messageforge-phase1-technical-handoff.md` -- section 8 (security model)
- `MessageForge.Documentation/handoff/sf-data-model.md`

**Requires:** Stream A (Data Model). Stream B (EncryptionService -- referenced but not directly called by this stream).

**Specification:**

**Sharing Model Prerequisites:**
- `Messenger_Chat__c` OWD must be set to `Private` so that Apex Managed Sharing controls access.
- `Messenger_Message__c` and `Messenger_Attachment__c` are Master-Detail children of `Messenger_Chat__c`, so they inherit the parent's sharing.
- The share object is `tgint__Messenger_Chat__Share` (auto-created by Salesforce when OWD is Private on a custom object).

**`ChannelAccessService.cls`**
- `public without sharing class ChannelAccessService` (must be `without sharing` to manipulate share records regardless of running user's permissions)
- Core method: `public static void recalculateSharing(Set<Id> channelIds)`:
  1. Query all `Channel_User_Access__c` where `Channel__c IN :channelIds` and `Is_Active__c = true`.
  2. Query all `Messenger_Chat__c` where `Channel__c IN :channelIds`.
  3. Delete all existing `Messenger_Chat__Share` records for these chats where `RowCause = Schema.Messenger_Chat__Share.RowCause.Manual` (Apex Managed Sharing uses 'Manual' row cause in managed packages, or use a custom row cause if defined).
  4. For each `Channel_User_Access__c` row:
     - If `Assignee_Type__c = 'user'` and `User__c` is populated: create `Messenger_Chat__Share` for each chat in that channel, with `UserOrGroupId = User__c` and `AccessLevel` mapped from `Access_Level__c` ('owner' -> 'Edit', 'read_write' -> 'Edit', 'read_only' -> 'Read').
     - If `Assignee_Type__c = 'group'` and `Group_ID__c` is populated: create `Messenger_Chat__Share` with `UserOrGroupId = Group_ID__c`.
  5. Insert all share records in bulk.
  6. Insert audit log: `Event_Type__c = 'sharing_recalculated'` for each channel.

- Helper method: `public static void grantAccessToChat(Id chatId, Id channelId)`:
  - Called when a new chat is created (from inbound handler). Queries `Channel_User_Access__c` for the channel and creates share records for just this one chat.

- Helper method: `public static void revokeAccessToChat(Id chatId)`:
  - Deletes all `Messenger_Chat__Share` records for this chat. Used when a chat is archived.

**`ChannelAccessTrigger.trigger`**
- On `tgint__Channel_User_Access__c` after insert, after update, after delete.
- Collects the `Channel__c` IDs from the trigger records.
- Calls `ChannelAccessService.recalculateSharing(channelIds)`.
- Trigger handler class: `ChannelAccessTriggerHandler.cls`.
- Bulkified: operates on all records in the trigger batch.

**`ChannelCompromiseTrigger.trigger`**
- On `tgint__Messenger_Channel__c` after update.
- When `Is_Compromised__c` changes from `false` to `true`:
  1. Insert `Channel_Audit_Log__c` with `Event_Type__c = 'compromised'`, `Event_Detail__c = '{"action": "kill_switch_activated", "phase": "1"}'`.
  2. Phase 1 stub: log only. In Phase 2, this trigger will additionally signal the Go gateway to terminate the channel session.
  3. No webhook unregistration here (that is a manual action the admin takes after investigation).
- Trigger handler class: `ChannelCompromiseTriggerHandler.cls`.

**`ChannelAuditService.cls`**
- `public with sharing class ChannelAuditService`
- `public static void log(Id channelId, String eventType, String eventDetail)`:
  - Insert `Channel_Audit_Log__c` with `Channel__c = channelId`, `Event_Type__c = eventType`, `Event_Detail__c = eventDetail`, `Performed_By__c = UserInfo.getUserId()`, `Event_Timestamp__c = Datetime.now()`, `Source__c = 'salesforce'`.
- `public static void log(Id channelId, String eventType, Map<String, Object> eventDetailMap)`:
  - Overload that serializes the map to JSON for `Event_Detail__c`.
- `public static void logBulk(List<Channel_Audit_Log__c> logs)`:
  - Bulk insert for trigger contexts.

**Deliverables:**
```
force-app/main/default/classes/ChannelAccessService.cls (+ meta)
force-app/main/default/classes/ChannelAccessTriggerHandler.cls (+ meta)
force-app/main/default/classes/ChannelCompromiseTriggerHandler.cls (+ meta)
force-app/main/default/classes/ChannelAuditService.cls (+ meta)
force-app/main/default/triggers/ChannelAccessTrigger.trigger (+ meta)
force-app/main/default/triggers/ChannelCompromiseTrigger.trigger (+ meta)
```

**Acceptance criteria:**
1. `sf project deploy start --source-dir MessageForge.SalesForce/force-app --dry-run` passes.
2. All Apex classes and triggers compile without errors.
3. When a `Channel_User_Access__c` record is inserted for a user, `Messenger_Chat__Share` records are created for all chats in that channel.
4. When `Is_Compromised__c` is set to true, an audit log entry is created.
5. `ChannelAuditService.log()` correctly inserts audit records.

**Constraints:**
- Namespace prefix: `tgint__` for all custom objects/fields/events
- API version: `62.0`
- Package type: 2GP managed package
- Format: SFDX source format
- Target directory: `MessageForge.SalesForce/force-app/main/default/`
- `ChannelAccessService` must be `without sharing` (needs to manipulate share records)
- All trigger handlers must be bulkified (handle 200+ records per batch)
- No recursive trigger execution (use static Boolean flags in handler classes)

---

### Stream G -- Tests

You are a Salesforce Apex test engineer working on the MessageForge managed package. Your task is to write comprehensive test classes for all Apex code in Streams B through F, achieving 80%+ code coverage for every class and trigger, with bulk test scenarios (200 records) and negative test cases.

**Goal:** Deliver test classes for every Apex class and trigger in the package, mock callout classes for each provider, a test data factory, and achieve 80%+ code coverage across the entire package.

**Files to read first:**
- `MessageForge.SalesForce/CLAUDE.md`
- All Apex classes and triggers in `force-app/main/default/classes/` and `force-app/main/default/triggers/` (read them all to understand what needs testing)
- `MessageForge.Documentation/handoff/messageforge-phase1-technical-handoff.md` -- sections 4, 5, 8 (for expected behaviors)

**Requires:** All other streams (A through F) must be deployed. All classes and triggers must exist and compile.

**Specification:**

**`TestDataFactory.cls`**
- `@isTest public class TestDataFactory`
- Static methods to create test records for every custom object:
  - `createChannel(String channelType, String status)` -- creates `Messenger_Channel__c` with the given type and status.
  - `createTelegramBotConfig(Id channelId)` -- creates `Telegram_Bot_Config__c` with test data.
  - `createWhatsAppConfig(Id channelId)` -- creates `WhatsApp_Config__c`.
  - `createViberConfig(Id channelId)`, `createTwilioConfig(Id channelId)`, `createMessengerConfig(Id channelId)`, `createInstagramConfig(Id channelId)`.
  - `createTelegramApp()`, `createWhatsAppApp()`, `createTwilioApp()`.
  - `createChat(Id channelId, String title)` -- creates `Messenger_Chat__c`.
  - `createMessage(Id chatId, String direction, String messageType, String text)` -- creates `Messenger_Message__c`.
  - `createAttachment(Id messageId)` -- creates `Messenger_Attachment__c`.
  - `createTemplate(Id channelId, String name)` -- creates `Messenger_Template__c`.
  - `createChannelUserAccess(Id channelId, Id userId, String accessLevel)` -- creates `Channel_User_Access__c`.
  - `createAuditLog(Id channelId, String eventType)` -- creates `Channel_Audit_Log__c`.
  - `createLead(String phone)` -- creates Lead with the given phone.
  - `createContact(String phone)` -- creates Contact with the given phone.
  - All methods return the inserted record. All use `@isTest` data isolation.
  - `createBulkChats(Id channelId, Integer count)` -- creates `count` chats in bulk.
  - `createBulkMessages(Id chatId, Integer count)` -- creates `count` messages in bulk.

**Mock Callout Classes:**
- `TelegramCalloutMock.cls` -- implements `HttpCalloutMock`. Returns realistic Telegram API responses for `getMe`, `setWebhook`, `sendMessage`, `getFile`.
- `WhatsAppCalloutMock.cls` -- returns realistic Meta Graph API responses.
- `ViberCalloutMock.cls` -- returns Viber PA API responses.
- `TwilioCalloutMock.cls` -- returns Twilio API responses.
- `MessengerCalloutMock.cls` -- returns Facebook Messenger API responses.
- `InstagramCalloutMock.cls` -- returns Instagram API responses.
- `MultiCalloutMock.cls` -- implements `HttpCalloutMock` with a map of endpoint -> response pairs for tests that make multiple callouts.

**Test Classes (minimum required):**

1. **`EncryptionServiceTest.cls`**:
   - Test encrypt and decrypt roundtrip.
   - Test decrypt with wrong key throws exception.
   - Test with no active encryption key throws exception.

2. **`ChannelSetupControllerTest.cls`**:
   - Test `listChannelTypes()` returns all CMT records.
   - Test `createApp()` for Telegram, WhatsApp, Twilio types.
   - Test `createChannel()` creates channel + config.
   - Test `testConnection()` with mock callout for each provider.
   - Test `activateChannel()` updates status and creates audit log.
   - Test `registerWebhook()` with mock callout for each provider.
   - Test `toggleActive()`.
   - Test `softDeleteChannel()` and `hardDeleteChannel()`.
   - Test `hardDeleteChannel()` fails when chat children exist.
   - Test `listChannels()` with various filter combinations.
   - Test `getRecentAuditLogs()`.
   - Bulk test: create 200 channels in a loop.

3. **`MessengerInboundAPITest.cls`**:
   - Test `doPost()` for each of the 6 channel types with valid payloads.
   - Test HMAC validation failure returns 401.
   - Test inactive channel returns 410.
   - Test compromised channel returns 410.
   - Test `doGet()` for WhatsApp/Messenger/Instagram webhook verification.
   - Test new chat creation with Lead matching by phone.
   - Test new chat creation with Contact matching.
   - Test new chat creation as orphan (no match).
   - Test duplicate message dedup via `Message_External_ID__c`.
   - Test media download for small file (<5MB).
   - Test media too large graceful failure.
   - Bulk test: POST with 200 messages in a single WhatsApp batch.

4. **`HMACValidatorTest.cls`**:
   - Test valid HMAC for each provider type.
   - Test invalid HMAC throws exception for each provider type.

5. **Per-parser test classes** (`TelegramInboundParserTest.cls`, `WhatsAppInboundParserTest.cls`, `ViberInboundParserTest.cls`, `TwilioInboundParserTest.cls`, `MessengerInboundParserTest.cls`, `InstagramInboundParserTest.cls`):
   - Test text message parsing.
   - Test media message parsing (photo, document, video).
   - Test empty/malformed payload handling.
   - For WhatsApp: test status update parsing (delivery receipts).

6. **`MessengerOutboundServiceTest.cls`**:
   - Test send for each channel type with mock callouts.
   - Test 4xx error handling (permanent failure).
   - Test 5xx error handling (transient failure).

7. **`MessengerOutboundQueueableTest.cls`**:
   - Test successful send.
   - Test retry on transient failure (verify re-enqueue).
   - Test max retry exhaustion.
   - Test kill switch blocks send.

8. **`MessengerControllerTest.cls`**:
   - Test `getChatList()` with various filters and pagination.
   - Test `getChatsForRecord()`.
   - Test `getRecentMessages()` with pagination.
   - Test `sendMessage()` inserts message and enqueues queueable.
   - Test `sendTemplate()` with valid template.
   - Test `sendTemplate()` with unapproved template fails.
   - Test `markAsRead()`.
   - Test `linkToRecord()`.
   - Test `getTemplates()`.
   - Test `getInboxChats()` for each tab filter.
   - Negative: `sendMessage()` to archived chat fails.
   - Negative: `sendMessage()` without access fails.

9. **`ChannelAccessServiceTest.cls`**:
   - Test sharing recalculation creates correct share records.
   - Test grant access to new chat.
   - Test revoke access.
   - Bulk test: 200 chats in a channel, trigger sharing recalc.

10. **`ChannelAccessTriggerTest.cls`**:
    - Test insert of `Channel_User_Access__c` triggers sharing recalc.
    - Test update triggers recalc.
    - Test delete triggers recalc.
    - Bulk test: insert 200 access records.

11. **`ChannelCompromiseTriggerTest.cls`**:
    - Test setting `Is_Compromised__c = true` creates audit log.
    - Test setting `Is_Compromised__c = false` (no action).

12. **`DeliveryStatusTriggerTest.cls`**:
    - Test status change publishes platform event.
    - Test no change does not publish.

13. **`InboundMessageTriggerTest.cls`**:
    - Test inbound message sends notification to assigned user.
    - Test inbound message with no assigned user (no notification, no error).
    - Bulk test: 200 messages.

14. **`ChannelAuditServiceTest.cls`**:
    - Test `log()` creates audit record.
    - Test `logBulk()` creates multiple records.

15. **`MediaDownloadServiceTest.cls`**:
    - Test download and attach for small file.
    - Test media too large graceful fail.

**Deliverables:**
```
force-app/main/default/classes/TestDataFactory.cls (+ meta)
force-app/main/default/classes/TelegramCalloutMock.cls (+ meta)
force-app/main/default/classes/WhatsAppCalloutMock.cls (+ meta)
force-app/main/default/classes/ViberCalloutMock.cls (+ meta)
force-app/main/default/classes/TwilioCalloutMock.cls (+ meta)
force-app/main/default/classes/MessengerCalloutMock.cls (+ meta)
force-app/main/default/classes/InstagramCalloutMock.cls (+ meta)
force-app/main/default/classes/MultiCalloutMock.cls (+ meta)
force-app/main/default/classes/EncryptionServiceTest.cls (+ meta)
force-app/main/default/classes/ChannelSetupControllerTest.cls (+ meta)
force-app/main/default/classes/MessengerInboundAPITest.cls (+ meta)
force-app/main/default/classes/HMACValidatorTest.cls (+ meta)
force-app/main/default/classes/TelegramInboundParserTest.cls (+ meta)
force-app/main/default/classes/WhatsAppInboundParserTest.cls (+ meta)
force-app/main/default/classes/ViberInboundParserTest.cls (+ meta)
force-app/main/default/classes/TwilioInboundParserTest.cls (+ meta)
force-app/main/default/classes/MessengerInboundParserTest.cls (+ meta)
force-app/main/default/classes/InstagramInboundParserTest.cls (+ meta)
force-app/main/default/classes/MessengerOutboundServiceTest.cls (+ meta)
force-app/main/default/classes/MessengerOutboundQueueableTest.cls (+ meta)
force-app/main/default/classes/MessengerControllerTest.cls (+ meta)
force-app/main/default/classes/ChannelAccessServiceTest.cls (+ meta)
force-app/main/default/classes/ChannelAccessTriggerTest.cls (+ meta)
force-app/main/default/classes/ChannelCompromiseTriggerTest.cls (+ meta)
force-app/main/default/classes/DeliveryStatusTriggerTest.cls (+ meta)
force-app/main/default/classes/InboundMessageTriggerTest.cls (+ meta)
force-app/main/default/classes/ChannelAuditServiceTest.cls (+ meta)
force-app/main/default/classes/MediaDownloadServiceTest.cls (+ meta)
```

**Acceptance criteria:**
1. `sf project deploy start --source-dir MessageForge.SalesForce/force-app --dry-run` passes.
2. All test classes compile without errors.
3. `sf apex run test --test-level RunLocalTests --code-coverage --wait 30` reports 80%+ overall code coverage.
4. Every Apex class has at least one corresponding test class.
5. Every test class has at least one positive test and one negative test.
6. Bulk test methods test with 200 records minimum.
7. All callout tests use `HttpCalloutMock` implementations.

**Constraints:**
- Namespace prefix: `tgint__` for all custom objects/fields/events
- API version: `62.0`
- Package type: 2GP managed package
- Format: SFDX source format
- Target directory: `MessageForge.SalesForce/force-app/main/default/`
- All test classes must be annotated with `@isTest`
- All test methods must be annotated with `@isTest` or use `testMethod` keyword
- All test data must be created within the test (no `SeeAllData=true`)
- Test methods must use `Test.startTest()` / `Test.stopTest()` around the code under test
- Mock callout classes must implement `HttpCalloutMock` interface
- Test class names must end with `Test` suffix
