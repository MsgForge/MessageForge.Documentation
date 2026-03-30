# Salesforce Data Model Reference

> **Canonical source:** `MessageForge.Salesforce/docs/reference/sf-data-model-v11.md` contains the full data model specification with all field definitions, encryption strategy, channel setup lifecycle, and resolved decisions.
>
> This file is a quick-reference summary.

## Custom Objects (21 + 2 Protected CMTs)

```
Channel_Type__mdt                  -- Package CMT. Channel type reference (read-only for customer)
├── DeveloperName                  -- Telegram_Bot, Telegram_User, WhatsApp_Cloud, etc.
├── Messenger__c                   -- Telegram, WhatsApp, Viber, Twilio, Vonage, Bird, Apple
├── Protocol__c                    -- bot_api, mtproto, cloud_api, rest, msp_api
├── Config_Object_Name__c          -- API name of child config object
└── Supports_Outbound__c           -- Can agent initiate new chat

Messenger_Channel__c               -- Base channel object (fully platform-agnostic)
├── Channel_Type_API_Name__c       -- DeveloperName of Channel_Type__mdt
├── Display_Name__c
├── Status__c                      -- draft | active | suspended | archived
├── Session_Status__c              -- online | offline | connecting | auth_required | mfa_required | flood_wait | error
├── Session_Error_Detail__c
├── Session_Last_Heartbeat__c
├── Active__c                      -- Admin on/off toggle
├── Is_Compromised__c              -- Security kill switch
├── External_Account_ID__c         -- Bot username, phone, Business ID
├── Go_Connection_Ref__c           -- PostgreSQL connection ID
├── Last_Connected_At__c           -- Last successful connection
└── Credential_Rotated_At__c       -- Last credential rotation

Platform App Objects (account-level, shared across channels):
├── Telegram_App__c                -- my.telegram.org app (api_id + api_hash)
├── WhatsApp_App__c                -- Meta App (App ID + App Secret)
├── Twilio_App__c                  -- Twilio Account (Account SID + Auth Token)
├── Vonage_App__c                  -- Vonage Account (API Key + API Secret)
└── Bird_App__c                    -- Bird Organization

Platform Config Objects (1:1 Master-Detail to Channel):
├── Telegram_Bot_Config__c         -- Bot Token, Bot Username
├── Telegram_User_Config__c        -- Phone, 2FA Password, App Lookup
├── WhatsApp_Config__c             -- Phone Number ID, WABA ID, Access Token (AES-256)
├── Viber_Config__c                -- Auth Token, Bot Name
├── Twilio_Config__c               -- Phone, API Key, Messaging Service SID
├── Vonage_Config__c               -- Phone, Application ID, Private Key (AES-256)
├── Bird_Config__c                 -- Phone, Access Key, Workspace/Channel IDs
└── Apple_Messages_Config__c       -- Business ID, Intent ID, Registration Type

Channel_User_Access__c             -- Polymorphic access junction (User OR Group)
├── Channel__c                     -- Master-Detail → Channel
├── Assignee_Type__c               -- user | group
├── User__c                        -- Lookup → User (when type = user)
├── Group_ID__c                    -- Text(18) (when type = group)
├── Group_Name__c                  -- Denormalized group name for display
├── Access_Level__c                -- owner | read_write | read_only
├── Is_Primary_Assignee__c         -- Lead routing flag
└── Is_Active__c                   -- Temp disable

Messenger_Chat__c                  -- Conversation thread
├── Channel__c                     -- Lookup → Channel (survives deletion)
├── Lead__c                        -- Lookup → Lead (nullable, cross-channel)
├── Contact__c                     -- Lookup → Contact (nullable)
├── Assigned_User__c               -- Lookup → User
├── Chat_External_ID__c            -- Unique External ID
├── Chat_Title__c
├── Chat_Type__c                   -- private | group | supergroup | channel
├── Status__c                      -- open | pending | resolved | archived
├── Last_Message_At__c
├── Last_Message_Preview__c
└── Unread_Count__c

Messenger_Message__c               -- Individual message
├── Chat__c                        -- Master-Detail → Chat (cascade delete)
├── Message_External_ID__c         -- External ID (upsert dedup)
├── Sender_Name__c
├── Sender_External_ID__c
├── Message_Text__c                -- Long Text Area (32000)
├── Direction__c                   -- inbound | outbound
├── Message_Type__c                -- text | photo | video | voice | document | sticker | template | album
├── Template__c                    -- Lookup → Template (when type = template)
├── Media_URL__c                   -- Cloudflare R2 CDN URL (primary/single attachment shortcut)
├── Media_MIME_Type__c
├── Attachment_Count__c            -- Roll-Up from Messenger_Attachment__c
├── Delivery_Status__c             -- pending → sent → delivered → read → failed
├── Delivery_Error__c
├── Sent_At__c
├── Reply_To_External_ID__c        -- Thread parent (text, intentionally not Lookup)
└── Is_Edited__c

Messenger_Attachment__c            -- Media attachment (child of Message, multi-media)
├── Message__c                     -- Master-Detail → Message (cascade delete)
├── Media_URL__c                   -- Cloudflare R2 CDN URL (required)
├── Media_MIME_Type__c             -- MIME type
├── File_Name__c                   -- Original filename
├── File_Size__c                   -- Bytes
├── Sort_Order__c                  -- Position in album (1-based)
├── Thumbnail_URL__c               -- Preview image URL
├── Media_External_ID__c           -- Platform file ID
├── Duration_Seconds__c            -- For voice/video
├── Width__c                       -- Image/video pixels
└── Height__c                      -- Image/video pixels

Messenger_Template__c              -- Reusable message template (WhatsApp required)
├── Channel__c                     -- Lookup → Channel (nullable for org-wide)
├── Template_Name__c               -- Platform template name
├── Template_Language__c           -- BCP 47 (en_US, es, etc.)
├── Template_Body__c               -- Long Text with {{1}} placeholders
├── Template_Header__c             -- Optional header
├── Template_Footer__c             -- Optional footer
├── Template_Category__c           -- marketing | utility | authentication
├── Platform_Template_ID__c        -- Platform-assigned ID
├── Approval_Status__c             -- draft | pending | approved | rejected | paused | disabled
├── Rejection_Reason__c
├── Is_Active__c                   -- Admin toggle
├── Messenger__c                   -- WhatsApp | Viber | Twilio | etc.
├── Button_JSON__c                 -- JSON button definitions
├── Last_Synced_At__c
└── Usage_Count__c

Channel_Audit_Log__c               -- Security audit trail
├── Channel__c                     -- Lookup → Channel (survives deletion)
├── Event_Type__c                  -- credential_rotated, compromised, session_connected, etc.
├── Event_Detail__c                -- JSON payload
├── Performed_By__c                -- Lookup → User
├── Event_Timestamp__c
└── Source__c                      -- salesforce | go_middleware
```

## Protected Custom Metadata (package-internal, invisible to customer)

```
tgint__Encryption_Key__mdt         -- AES-256 key for Apex-level encryption
├── Key_Value__c                   -- Base64 key
└── Is_Active__c                   -- Key rotation support

tgint__Apple_MSP__mdt              -- Our Apple Messages MSP credentials
├── MSP_ID__c                      -- UUID
├── MSP_Secret__c                  -- Base64 secret
└── Endpoint_URL__c                -- Apple API endpoint
```

## Platform Events

```
tgint__Inbound_Message__e          -- Go → SF: new inbound messages
├── Text__c
├── Chat_SF_ID__c
├── Sender_Name__c
├── Sender_External_ID__c
├── Message_Type__c
├── Media_URL__c
├── Media_MIME_Type__c
└── Sent_At__c

tgint__Session_Status__e           -- Go → SF: channel session lifecycle
├── Channel_SF_ID__c
├── Status__c                      -- online | offline | connecting | auth_required | mfa_required | flood_wait | error
└── Error_Detail__c

tgint__Message_Delivery_Status__e  -- Go → SF: outbound delivery status
├── Message_External_ID__c
├── Status__c                      -- DELIVERED | FAILED | RETRYING | READ
├── Error_Code__c
├── Error_Detail__c
└── Timestamp__c
```

## Relationship Map

```
Channel_Type__mdt ──(text ref)──► Messenger_Channel__c ──Lookup(Owner)──► User
                                        │
    ┌───────────────────────────────────┤
    │ MD                                │ MD
    ▼                                   ▼
  Config objects (1:1)              Channel_User_Access__c
    │                                   │
    │ Lookup (some configs)             ├── Lookup → User (when type=user)
    ▼                                   └── Text → Group ID (when type=group)
  App objects
    │
    │ Lookup
    ▼
  User (Manager)

Messenger_Channel__c
    │
    │ Lookup (Chat survives Channel deletion)
    ▼
Messenger_Chat__c ──Lookup──► Lead (nullable, many:1)
    │              ──Lookup──► Contact (nullable)
    │              ──Lookup──► User (Assigned)
    │
    │ Master-Detail (cascade delete, roll-up)
    ▼
Messenger_Message__c ──Lookup──► Messenger_Template__c
    │
    │ Master-Detail (cascade delete, roll-up)
    ▼
Messenger_Attachment__c

Messenger_Template__c ──Lookup──► Messenger_Channel__c (nullable for org-wide)

Channel_Audit_Log__c ──Lookup──► Channel (survives deletion)
                     ──Lookup──► User (Performed_By)
```

## Managed Package Structure (2GP)

```
Managed Package: "MessageForge" (namespace: tgint)
├── Custom Objects:
│   ├── tgint__Messenger_Channel__c
│   ├── tgint__Telegram_App__c / WhatsApp_App__c / Twilio_App__c / Vonage_App__c / Bird_App__c
│   ├── tgint__Telegram_Bot_Config__c / Telegram_User_Config__c / WhatsApp_Config__c
│   ├── tgint__Viber_Config__c / Twilio_Config__c / Vonage_Config__c / Bird_Config__c / Apple_Messages_Config__c
│   ├── tgint__Channel_User_Access__c
│   ├── tgint__Messenger_Chat__c
│   ├── tgint__Messenger_Message__c
│   ├── tgint__Messenger_Attachment__c
│   ├── tgint__Messenger_Template__c
│   └── tgint__Channel_Audit_Log__c
├── Custom Metadata Types:
│   ├── tgint__Channel_Type__mdt (package-deployed, read-only)
│   ├── tgint__Encryption_Key__mdt (Protected)
│   └── tgint__Apple_MSP__mdt (Protected)
├── Platform Events:
│   ├── tgint__Inbound_Message__e
│   ├── tgint__Session_Status__e
│   └── tgint__Message_Delivery_Status__e
├── Apex Classes:
│   ├── MessengerInboundAPI.cls (@RestResource — receives webhooks)
│   ├── MessengerOutboundService.cls (callouts to Go)
│   ├── MessengerController.cls (LWC backend — chats, messages)
│   ├── ChannelSetupController.cls (LWC backend — channel wizard)
│   ├── ChannelAccessService.cls (Apex Managed Sharing)
│   ├── EncryptionService.cls (AES-256 encrypt/decrypt)
│   ├── CentrifugoTokenController.cls (JWT generation)
│   └── HMACValidator.cls (shared verification)
├── Apex Triggers:
│   ├── InboundMessageTrigger.trigger (PE → record creation)
│   ├── SessionStatusTrigger.trigger (PE → channel status)
│   ├── DeliveryStatusTrigger.trigger (PE → delivery status)
│   ├── ChannelAccessTrigger.trigger (sharing recalculation)
│   └── ChannelCompromiseTrigger.trigger (kill switch → Go notification)
├── Lightning Web Components:
│   ├── channelSetupWizard (admin — guided channel creation)
│   ├── messengerChat (main chat UI)
│   ├── messengerLiveChat (Centrifugo wrapper)
│   ├── messengerMediaViewer (R2/CDN media lightbox)
│   └── centrifugoClient (service module)
├── Permission Sets:
│   ├── tgint__Messenger_Admin (full CRUD on channels, configs, apps, audit)
│   ├── tgint__Messenger_Agent (chat + message CRUD, limited channel read)
│   └── tgint__Messenger_Viewer (read-only on chats, messages, audit)
├── Connected App:
│   └── Messenger_Go_Server (OAuth 2.0)
└── CSP Trusted Sites:
    ├── Centrifugo WebSocket domain
    └── Cloudflare Worker CDN domain
```
