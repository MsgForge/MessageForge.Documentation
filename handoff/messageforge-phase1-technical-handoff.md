# MessageForge Phase 1 — Technical Handoff for LWC Design

**Audience:** Agent responsible for designing Lightning Web Components and producing finalized technical requirements for the developer team.

**Purpose:** Define the Phase 1 (native-only MVP) architecture in enough detail that the LWC design work can proceed against a stable contract, while flagging the surfaces where LWC decisions may legitimately push back and force architectural changes before requirements are frozen.

**Status:** Draft for review. Not frozen. Sections marked **[OPEN]** are explicitly negotiable based on LWC design feedback.

---

## 1. Phase 1 scope

### In scope (native Apex/LWC only, zero external infrastructure)

- **Channels:** Telegram Bot API, WhatsApp Cloud API, Viber Bot API, Twilio SMS, Facebook Messenger, Instagram Direct.
- **Inbound:** webhooks land on Apex REST endpoints exposed via Salesforce Sites (Guest User), HMAC-verified, persisted to `tgint__Messenger_Message__c`.
- **Outbound:** Apex callouts via Named Credentials to each provider's REST API.
- **Real-time UI:** Platform Events (`tgint__Inbound_Message__e`, `tgint__Message_Delivery_Status__e`) → LWC via `lightning/empApi`.
- **Media:** images/audio/video/documents up to the Apex async heap limit (~12 MB). Stored as `ContentVersion` records, linked to `Messenger_Message__c` via `Messenger_Attachment__c`.
- **Channel setup:** admin LWC wizard creates a `Messenger_Channel__c` + matching `*_Config__c` record, registers the webhook with the provider, validates connectivity.
- **Conversation routing:** chats linked to `Lead`/`Contact` by lookup; assignment via `Channel_User_Access__c` + `Messenger_Chat__c.Assigned_User__c`.
- **Templates:** WhatsApp template sync and send (required by Meta for outside-24h-window messages).
- **Security:** AES-256 encryption of stored credentials, Apex Managed Sharing on chats, audit log on every credential/security event.

### Out of scope — explicitly deferred to Phase 2

- **Telegram User accounts (MTProto).** The data model already contains `Telegram_User_Config__c` and `Channel_Type__mdt` rows for it; in Phase 1 these are marked `Status__c = 'draft'` and the channel type is hidden in the setup wizard.
- **Large media** (Telegram Bot files >12 MB, WhatsApp documents >12 MB). Provider API responses that exceed the Apex async heap will be persisted as `Messenger_Message__c` with `Media_URL__c = null`, `Last_Error__c = 'media_too_large_phase2'`, and a placeholder UI state. **No partial download attempts.**
- **Go middleware.** The fields `Go_Connection_Ref__c`, `Middleware_Config__mdt`, `tgint__Session_Status__e`, and the `Source__c = 'go_middleware'` audit value remain in the schema but are unused in Phase 1. Do not remove them — Phase 2 will use them as-is.
- **Centrifugo / external real-time bus.** Phase 1 uses Platform Events + empApi exclusively.
- **Apple Messages for Business.** `Apple_Messages_Config__c` and `Apple_MSP__mdt` stay in the schema but the channel type is disabled in the wizard.
- **Vonage / Bird.** Same — schema present, wizard hides them.

The data model in `sf-data-model-v11.md` is the canonical source. Phase 1 uses a subset of it; nothing should be deleted.

---

## 2. Architecture overview

```
┌────────────────────────────────────────────────────────────────────┐
│  Provider (Telegram / WhatsApp / Viber / Twilio / Meta)            │
└──────────────────────────┬─────────────────────────────────────────┘
                           │ HTTPS webhook (POST)
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│  Salesforce Site (public, Guest User profile)                      │
│  ─ /services/apexrest/tgint/webhook/{channelType}/{channelId}     │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│  MessengerInboundAPI.cls (@RestResource)                           │
│   1. HMACValidator.verify(rawBody, headers, channelType)          │
│   2. Parse provider payload → normalized DTO                       │
│   3. Upsert Chat by Chat_External_ID__c                            │
│   4. Insert Message + Attachments                                  │
│   5. EventBus.publish(tgint__Inbound_Message__e)                  │
│   6. Return 200 (fast — <5s target)                                │
└──────────────────────────┬─────────────────────────────────────────┘
                           │ Platform Event
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│  LWC (messengerChat) via lightning/empApi                          │
│   ─ Subscribed to /event/tgint__Inbound_Message__e                 │
│   ─ Refreshes the open chat view in real time                      │
└────────────────────────────────────────────────────────────────────┘

Outbound flow:
LWC → Apex @AuraEnabled (MessengerController.sendMessage)
     → MessengerOutboundService.send(channel, message)
     → HTTP callout via Named Credential (per channel)
     → Insert Messenger_Message__c with Direction='outbound', Delivery_Status='pending'
     → Provider response updates Delivery_Status to 'sent'
     → Provider webhook (later) updates to 'delivered' / 'read' / 'failed'
```

**Key design constraint:** Phase 1 has *no* server outside Salesforce. Every byte of state lives in the Salesforce org. This is a feature, not a limitation — it's the data residency story we sell.

---

## 3. Data model — Phase 1 working subset

The full model has 21 custom objects, 4 protected CMTs, and 3 platform events. Phase 1 actively uses the following.

### Always-on objects

| Object | Phase 1 role |
|---|---|
| `Channel_Type__mdt` | Reference table; only Phase 1 channel types are surfaced in the wizard |
| `Messenger_Channel__c` | One per active integration. `Status__c`, `Active__c`, `Is_Compromised__c` are the runtime flags |
| `Telegram_App__c` / `WhatsApp_App__c` / `Twilio_App__c` | Account-level credentials, shared across channels |
| `Telegram_Bot_Config__c` / `WhatsApp_Config__c` / `Viber_Config__c` / `Twilio_Config__c` | 1:1 with channel; holds the channel-specific settings |
| `Channel_User_Access__c` | Drives sharing — who can see/send on this channel |
| `Messenger_Chat__c` | Conversation thread, linked to `Lead`/`Contact` |
| `Messenger_Message__c` | Individual message, master-detail to Chat |
| `Messenger_Attachment__c` | 0..N media items per message |
| `Messenger_Template__c` | WhatsApp templates (synced from Meta) |
| `Channel_Audit_Log__c` | Every credential/security event |

### Phase 1 platform events

| Event | Direction | Trigger |
|---|---|---|
| `tgint__Inbound_Message__e` | Apex publish → LWC subscribe | Fires after every successful inbound webhook insert |
| `tgint__Message_Delivery_Status__e` | Apex publish → LWC subscribe | Fires when an outbound message's delivery status changes |

`tgint__Session_Status__e` exists in the schema but is **not used in Phase 1** (it's the Phase 2 Go gateway heartbeat channel).

### Phase 1 fields with special semantics

- `Messenger_Message__c.Last_Error__c` — populated when a media file exceeds Phase 1 limits, set to a stable enum string (`media_too_large_phase2`, `media_unsupported_format`, `provider_error`, etc.). LWC must render this gracefully.
- `Messenger_Channel__c.Session_Status__c` — in Phase 1, only the values `online`, `offline`, `auth_required`, `error` are used. The MFA/flood-wait values are Phase 2 (MTProto).
- `Channel_Audit_Log__c.Source__c` — always `'salesforce'` in Phase 1.

### Lead/Contact linkage rule [OPEN]

The current model permits a `Messenger_Chat__c` to link to *either* a `Lead` or a `Contact` (both nullable). The matching strategy in Phase 1 is:

1. **Inbound message arrives** with a sender external ID (phone number for WhatsApp/SMS/Viber, Telegram user ID for TG Bot, PSID for Messenger).
2. **Lookup existing chat** by `Channel__c` + `Chat_External_ID__c`. If found, reuse — Lead/Contact already linked.
3. **If new chat:** SOQL on `Lead` and `Contact` by phone (normalized to E.164) or by a custom external ID field. First match wins. If both match, prefer `Contact`.
4. **If no match:** chat is created with both lookups null. Agent manually links from the LWC.

This rule needs LWC design feedback because it has UX consequences (orphan chats, manual linking flow, conflict resolution when the same phone exists on multiple records). **The LWC agent should propose the resolution UI and may push back on the matching algorithm.**

---

## 4. Inbound message flow (per channel)

All channels follow the same skeleton; the differences are in HMAC verification and payload parsing.

### Generic skeleton

```apex
@RestResource(urlMapping='/webhook/*')
global class MessengerInboundAPI {
    @HttpPost
    global static void doPost() {
        RestRequest req = RestContext.request;
        // 1. Extract channelType + channelId from URI
        // 2. Load Messenger_Channel__c (one SOQL) — skip if Active__c=false or Is_Compromised__c=true
        // 3. HMACValidator.verify(req.requestBody, req.headers, channel)  → throws on failure
        // 4. Dispatch to channel-specific parser → returns NormalizedInboundDTO
        // 5. Upsert Chat (by Chat_External_ID__c)
        // 6. Insert Message + Attachments (single DML where possible)
        // 7. EventBus.publish(buildPlatformEvent(...))
        // 8. Return 200 OK with empty body
    }

    @HttpGet
    global static void doGet() {
        // Used by WhatsApp Cloud API for the verify_token challenge
    }
}
```

### Per-channel specifics

| Channel | Webhook URL pattern | Verification | GET handler? | Payload size | Notes |
|---|---|---|---|---|---|
| **Telegram Bot** | `/webhook/telegram_bot/{channelId}` | Header `X-Telegram-Bot-Api-Secret-Token` (set during `setWebhook`) | No | up to ~1 MB | Must `setWebhook` with the secret on channel activation |
| **WhatsApp Cloud** | `/webhook/whatsapp_cloud/{channelId}` | HMAC-SHA256 of body using App Secret, header `X-Hub-Signature-256` | **Yes** — `hub.challenge` echo | up to 3 MB (Meta limit) | Bundles status updates and inbound messages in same payload |
| **Viber Bot** | `/webhook/viber/{channelId}` | HMAC-SHA256 of body using auth token, header `X-Viber-Content-Signature` | No | small | Must respond <5s or Viber retries |
| **Twilio SMS** | `/webhook/twilio/{channelId}` | HMAC-SHA1 of full URL + sorted POST params, header `X-Twilio-Signature` | No | small | URL must match exactly (including query string) for signature to validate |
| **Facebook Messenger** | `/webhook/messenger/{channelId}` | HMAC-SHA256 (same as WhatsApp), header `X-Hub-Signature-256` | **Yes** — verify token | up to 3 MB | 7-day reply window outside which messages must use message tags |
| **Instagram** | `/webhook/instagram/{channelId}` | HMAC-SHA256 (same as WhatsApp) | **Yes** — verify token | up to 3 MB | Requires Instagram Business account linked to FB Page |

### Webhook handler performance budget

| Step | Target |
|---|---|
| HMAC verification | <50 ms |
| SOQL (channel + chat lookup) | <200 ms |
| DML (chat upsert + message insert + attachments) | <500 ms |
| Platform Event publish | <100 ms |
| **Total round trip** | **<2 s (hard ceiling 5 s)** |

If we exceed 5 s, providers retry, and we get duplicate-message bugs. The handler must not do anything synchronous beyond persistence + PE publish. Any heavy work (template parsing, AI, lead matching) is deferred to async triggers on the inserted records.

---

## 5. Outbound message flow

```
LWC sendMessage()
  └─► @AuraEnabled MessengerController.sendMessage(chatId, text, attachmentIds)
        ├─► Validate user has access via Channel_User_Access__c
        ├─► Insert Messenger_Message__c (Direction='outbound', Delivery_Status='pending')
        ├─► Enqueue MessengerOutboundQueueable
        └─► Return new message Id to LWC for optimistic rendering

MessengerOutboundQueueable.execute()
  ├─► Load message + chat + channel + config
  ├─► HTTP callout via Named Credential → provider API
  ├─► On 2xx: update Delivery_Status='sent', Sent_At=now
  ├─► On 4xx: update Delivery_Status='failed', Last_Error=parsed error
  ├─► On 5xx: requeue with backoff (max 3 attempts)
  └─► Publish tgint__Message_Delivery_Status__e
```

**Why queueable instead of synchronous callout from `sendMessage`:** the LWC must return immediately for responsive UX. Optimistic rendering shows the message in the conversation with a "sending" indicator, then transitions to "sent"/"failed" via the platform event subscription.

**Named Credentials:** one per channel instance. Created by the channel setup wizard. Phase 1 uses External Credentials with Custom auth provider for Telegram (bot token in URL path) and WhatsApp (Bearer token), per-request HMAC for Twilio, custom header for Viber.

**Rate limiting:** Phase 1 relies on per-channel queueable serialization (one queueable chain per channel). At MVP volumes this is sufficient; if a channel approaches its provider rate limit, we add token-bucket logic on the queueable. **No rate limiting in v1 release** — flagged as a known limitation.

---

## 6. Real-time UI propagation

### The flow

1. Inbound webhook handler publishes `tgint__Inbound_Message__e` synchronously after the DML commit.
2. LWC `messengerChat` subscribes via `lightning/empApi` to `/event/tgint__Inbound_Message__e` on connect.
3. On event receipt, LWC checks if the event's `Chat_SF_ID__c` matches the currently-open chat. If yes, append the message to the visible conversation. If no, increment an unread badge on the chat list.
4. Outbound delivery status updates flow the same way via `tgint__Message_Delivery_Status__e`.

### Latency expectations

- Platform Event publish → empApi delivery: typically 200 ms – 1 s.
- Worst case (CometD reconnect after sleep): up to 5 s.
- This is **good enough for a chat UI** at MVP volumes but is *not* WhatsApp-Web-grade real time.

### Allocation budget

Enterprise edition gets 10K external Platform Event deliveries per 24h. Each delivered event to each subscribed LWC client counts. With 10 concurrent agents, the budget burns at 10× the inbound message rate. **Hard cap for Phase 1 with default Enterprise allocations: ~1,000 inbound messages per day per org.**

For higher-volume customers, options are:
1. Require Unlimited edition (50K daily allocation).
2. Sell Platform Event allocation add-ons.
3. **[OPEN]** LWC-side polling fallback for high-volume customers (every 10 s when actively viewing a chat). This is a workaround, not pretty, but it caps platform event consumption.

**This is the single most important constraint for the LWC agent to understand.** The real-time UX has a hard ceiling that may force a "high-volume mode" toggle at the LWC level.

---

## 7. Media handling — Phase 1 limits

### What works

| Scenario | Path |
|---|---|
| Image ≤5 MB | Sync Apex callout downloads from provider, creates `ContentVersion`, links via `ContentDocumentLink` to `Messenger_Message__c`, populates `Messenger_Attachment__c` |
| Image/audio/video 5–12 MB | Same flow but in async context (Queueable enqueued from webhook handler) |
| Document ≤12 MB | Same as above |
| Voice note ≤5 MB | Sync path; metadata fields (`Duration_Seconds__c`) populated from provider payload |
| Telegram album (multiple photos) | Each photo → one `Messenger_Attachment__c` with `Sort_Order__c` set |

### What fails gracefully

| Scenario | Behavior |
|---|---|
| Provider says file is >12 MB | Message inserted with `Media_URL__c=null`, `Last_Error__c='media_too_large_phase2'`, `Message_Type__c` set to actual type (photo/video/document) |
| Download succeeds but ContentVersion insert fails | Message inserted, `Last_Error__c='content_version_failed'`, retried by async job up to 3× |
| Unknown MIME type | Stored as `document` with original MIME preserved |

### LWC implications

- The chat UI must render a "media unavailable in MVP — upgrade for large file support" placeholder for `Last_Error__c='media_too_large_phase2'`. This is **the main visible Phase 1 limitation** and the primary upsell hook for Phase 2. The LWC agent should treat this as a first-class UI state, not an error.
- The media viewer LWC (`messengerMediaViewer`) opens the `ContentVersion` via the `/sfc/servlet.shepherd/version/download/{Id}` URL pattern (per ADR-20).
- Image thumbnails: Phase 1 does not generate thumbnails. The full image is loaded inline. **[OPEN]** — the LWC agent may push for a thumbnail-generation step (would require an async job using a JavaScript canvas in the LWC, or a separate Queueable doing image resize via a third-party Apex library, or just lazy-loading the original).

---

## 8. Security model

### Credential storage

- All provider tokens (`WhatsApp_Config__c.Access_Token`, `Telegram_Bot_Config__c.Bot_Token`, etc.) stored AES-256 encrypted via `EncryptionService.cls`, key in `tgint__Encryption_Key__mdt` (Protected CMT, invisible to subscriber org admins).
- Apex Crypto class provides AES-CBC. Key rotation supported via `Is_Active__c` flag on multiple key records (decrypt with any active, encrypt with newest).
- **Never** stored in custom settings, environment variables, or named credentials' visible fields. Named Credentials are created per-channel at activation time using the decrypted token.

### Webhook authentication

- Salesforce Site Guest User has profile permissions for *only* the `MessengerInboundAPI` Apex class. No object access — all DML happens in `without sharing` Apex.
- HMAC verification is mandatory before any DML. Failure logs to `Channel_Audit_Log__c` with `Event_Type__c='hmac_verification_failed'` and returns 401.
- Replay protection: Phase 1 uses per-channel monotonic message IDs from providers (`update_id` for Telegram, message timestamps for others) and dedupes via `Message_External_ID__c` upsert. Not a perfect replay defense but sufficient against accidental retries.

### Sharing

- Apex Managed Sharing on `Messenger_Chat__c` based on `Channel_User_Access__c` rows.
- `ChannelAccessTrigger` recalculates sharing on `Channel_User_Access__c` insert/update/delete.
- Messages and attachments inherit chat sharing via master-detail.
- **Group access** (when `Assignee_Type__c='group'`) uses standard Group ID sharing — no extra logic needed beyond enumerating group members at recalc time.

### Kill switch

- `Messenger_Channel__c.Is_Compromised__c = true` causes:
  - All webhook handlers to return 410 immediately (no DML)
  - All outbound queueables to abort
  - Audit log entry
  - Phase 2 will additionally signal the Go gateway via `ChannelCompromiseTrigger`; in Phase 1 this trigger is a no-op stub

---

## 9. LWC surfaces required (high-level only — to be designed)

The LWC agent owns the detailed design of these. Listed here only so the contract with Apex is clear.

### `channelSetupWizard` (admin)

- Multi-step: pick channel type → enter app credentials → enter channel-specific config → test connectivity → activate.
- Backed by `ChannelSetupController.cls` (`@AuraEnabled` methods: `listChannelTypes`, `createApp`, `createChannel`, `testConnection`, `activateChannel`, `registerWebhook`).
- Must handle WhatsApp's two-step flow (verify business → register phone number).
- Must call provider APIs to register the webhook URL during activation, with `setWebhook` for Telegram, the Meta Graph API for WhatsApp/Messenger/Instagram, etc.

**[OPEN]** — How the wizard handles re-entry after a failed setup is a UX call. The agent should propose either "resume from last completed step" or "always start fresh, idempotent operations".

### `messengerChat` (agent/main UI)

- Three-pane layout (channel list / chat list / message thread) is the assumed default but **not mandated**. The agent should propose a layout that works on both desktop and tablet.
- Subscribes to `tgint__Inbound_Message__e` and `tgint__Message_Delivery_Status__e`.
- Sends via `MessengerController.sendMessage`.
- Must render:
  - Text messages (inbound and outbound, with delivery status indicators)
  - Image / video / audio / document attachments (≤12 MB)
  - Media unavailable placeholder (for `Last_Error__c='media_too_large_phase2'`)
  - WhatsApp templates picker for outbound when outside the 24h window
  - Reply-to threading (using `Reply_To_External_ID__c`)
  - Edit indicator (`Is_Edited__c`)
- **[OPEN]** — Channel-switcher UX. A single conversation may span channels if a Lead has both WhatsApp and Telegram. Whether the LWC merges these into one timeline or shows them as separate chats is a major UX call with data model consequences.

### `messengerMediaViewer`

- Lightbox for inline media. Opens `ContentVersion` URLs.
- **[OPEN]** — Whether to support download, share, or annotation actions.

### Lead/Contact embedded view

- A condensed `messengerChat` variant that lives on the Lead/Contact record page, filtered to chats linked to that record.
- **[OPEN]** — Whether this is a separate LWC or a `@api` mode of `messengerChat`.

---

## 10. Open design questions for the LWC agent

These are the questions whose answers may **change the architecture or data model** before requirements freeze. Each one needs a decision before developer handoff.

1. **Cross-channel timeline merging.** If a Lead has 3 chats across WhatsApp, Telegram, and SMS, does the agent see one unified timeline or three threads? Unification needs a `Conversation__c` parent object above `Messenger_Chat__c` (not in current model). Three-threaded view needs no schema change but worse UX.

2. **Lead/Contact matching algorithm.** The Phase 1 default (phone-based first-match) may produce wrong links and orphan chats. The LWC agent should specify the conflict-resolution UI and confirm the matching rule, or propose an alternative (e.g., always create as orphan, agent manually links).

3. **High-volume mode.** Phase 1 caps at ~1,000 inbound msgs/day on default Enterprise allocations because of Platform Event delivery limits. Does the LWC need a fallback polling mode for higher-volume orgs? If yes, the chat list and thread views need pull endpoints in `MessengerController`.

4. **Channel multiplexing in the chat list.** If an agent has access to 5 channels with 200 active chats each, how does the chat list scale? Pagination, virtual scrolling, search, filters — all LWC-level decisions that drive `MessengerController` query API design.

5. **Optimistic outbound rendering.** Confirmed pattern is "insert pending → return Id → LWC shows immediately → PE updates status". The agent should validate this works for their UI design or propose an alternative (e.g., synchronous send for slow-volume cases).

6. **Template picker UX.** WhatsApp templates have variables (`{{1}}`, `{{2}}`), categories, languages, and approval states. The LWC needs to handle variable substitution UI. The current `Messenger_Template__c.Template_Body__c` schema may need additional fields (e.g., a JSON variable schema) — flag if so.

7. **Channel setup wizard re-entry.** Already mentioned. Affects whether `Messenger_Channel__c.Status__c='draft'` rows accumulate or get cleaned up.

8. **Notification of new messages when LWC is not open.** Salesforce native bell notifications? Custom notifications via `CustomNotificationType`? Email digest? **None** in MVP? The PE → LWC path only fires for users actively viewing the LWC.

9. **Mobile.** Salesforce mobile app support. The data model is fine; the LWC layout decision is the constraint. **[OPEN]** — is mobile in Phase 1 scope at all?

10. **Media thumbnails.** Already mentioned. Decision needed before media storage code is written.

---

## 11. Areas where LWC design may force architectural changes

These are the things to watch for. If the LWC agent's answers to section 10 push us here, the architecture in this document is wrong and needs revision before the developer team starts.

| Trigger | Required architectural change |
|---|---|
| Cross-channel unified timeline | New `Conversation__c` parent object; `Messenger_Chat__c.Conversation__c` lookup; trigger to auto-link chats with matching Lead/Contact |
| High-volume polling mode | New `MessengerController` methods (`getRecentMessages(chatId, since)`, `getChatList(filters, page)`); polling rate management; Platform Event publish becomes optional |
| Notification system | `CustomNotificationType` deployment in package; trigger on `Messenger_Message__c` insert that notifies the assigned user |
| Mobile-first layout | LWC must run in Salesforce mobile app; constrains DOM size, gesture handling, and possibly forces a separate `messengerChatMobile` component |
| Template variable schema | New field on `Messenger_Template__c` (e.g., `Variable_Schema_JSON__c`); template sync logic must extract variables from Meta payload |
| Real-time read receipts | Outbound requires inbound webhook handling for `read` status events (already in scope for WhatsApp; needs confirmation for Messenger/Instagram) |
| Typing indicators | Requires sub-second latency, exceeds Platform Event capability — would force Phase 2 Centrifugo earlier than planned. **Recommendation: explicitly out of scope for Phase 1.** |

---

## 12. Governor limit budget (per webhook invocation)

| Limit | Used | Headroom |
|---|---|---|
| SOQL queries (100) | 3–5 | comfortable |
| DML statements (150) | 2–4 | comfortable |
| DML rows (10,000) | 1–20 (album) | comfortable |
| Callouts (100) | 1–2 (media download) | comfortable |
| Heap (6 MB sync / 12 MB async) | up to 12 MB on media | **tight — primary constraint** |
| CPU time (10 s sync / 60 s async) | 1–3 s typical | comfortable |
| Platform Events published per transaction | 1 | comfortable |

**Daily org-level limits to monitor:**
- Async Apex executions: 250K/day → comfortable up to ~200K msgs/day with media
- Platform Event deliveries to external clients: 10K/day Enterprise → **bottleneck at ~1K msgs/day with active LWC users**
- API calls (outbound from package context): subject to org limits, plan accordingly

---

## 13. Acceptance criteria for Phase 1

Phase 1 is "done" when:

1. A subscriber org admin can install the package and complete channel setup for at least Telegram Bot, WhatsApp Cloud, and Twilio SMS in under 15 minutes per channel.
2. Inbound messages arrive in `Messenger_Message__c` within 2 s of provider webhook delivery, p95.
3. Outbound messages send successfully via `MessengerController.sendMessage` and reach the provider within 5 s, p95.
4. The `messengerChat` LWC renders new inbound messages without manual refresh within 3 s, p95.
5. Media files up to 12 MB are stored as `ContentVersion` and viewable in the LWC.
6. Files larger than 12 MB produce the `media_too_large_phase2` placeholder without errors.
7. AppExchange Security Review passes.
8. No external infrastructure (no Go server, no Centrifugo, no S3) is required to run the package.
9. All credentials at rest are AES-256 encrypted.
10. The kill switch (`Is_Compromised__c`) blocks both inbound and outbound traffic within one transaction.

---

## 14. Next steps

1. **LWC agent** reviews this document and section 10 in particular. Produces answers/proposals for each open question.
2. **Architecture review** — answers may force changes to the data model, the controller API surface, or the platform event payloads. Update this document.
3. **Freeze contract** — once LWC and Apex agree on the controller method signatures, platform event payloads, and any data model deltas, this document becomes the input to developer task breakdown.
4. **Developer handoff** — break Phase 1 into work streams: (a) channel setup + credentials, (b) inbound webhook handlers per channel, (c) outbound queueables per channel, (d) LWC components, (e) sharing + security + audit, (f) Security Review prep.

---

**Document owner:** Architecture
**Review cycle:** before LWC design begins, and again before developer handoff
**Related:** `sf-data-model-v11.md` (canonical data model), Phase 2 architecture doc (TBD)
