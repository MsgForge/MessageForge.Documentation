# Salesforce Data Model Reference

## Custom Objects

```
Messenger_Connection__c        -- A configured messenger account
├── Connection_Type__c         -- 'mtproto_user' | 'bot_api' | 'whatsapp' (future)
├── Status__c                  -- 'active' | 'disconnected' | 'auth_required'
├── Messenger_Platform__c      -- 'telegram' | 'whatsapp' | 'messenger' (future)
├── External_Account_ID__c     -- Telegram user ID, phone number, bot username
└── Salesforce_User__c         -- Lookup to Salesforce User who owns this connection

Messenger_Chat__c              -- A Telegram chat/group/channel
├── Chat_External_ID__c        -- Telegram chat ID (unique)
├── Chat_Title__c
├── Chat_Type__c               -- 'private' | 'group' | 'supergroup' | 'channel'
├── Connection__c              -- Lookup to Messenger_Connection__c
└── Last_Message_At__c

Messenger_Message__c           -- An individual message
├── Message_External_ID__c     -- Telegram message ID (unique per chat)
├── Chat__c                    -- Lookup to Messenger_Chat__c
├── Sender_Name__c
├── Sender_External_ID__c
├── Message_Text__c            -- Long text area
├── Message_Type__c            -- 'text' | 'photo' | 'video' | 'voice' | 'document'
├── Media_URL__c               -- URL to Cloudflare R2/Worker (for media messages)
├── Sent_At__c                 -- DateTime from Telegram
├── Direction__c               -- 'inbound' | 'outbound'
├── Delivery_Status__c         -- 'pending' | 'delivered' | 'failed'
└── Protocol__c                -- 'mtproto' | 'bot_api'

Messenger_Event__c             -- Parsed events (activities, meetups, etc.)
├── Message__c                 -- Lookup to source Messenger_Message__c
├── Event_Title__c
├── Event_Date__c
├── Event_Location__c
├── Event_Description__c
└── Processing_Status__c       -- 'pending' | 'confirmed' | 'discarded'
```

## Platform Events

```
tgint__Inbound_Message__e      -- Go -> Salesforce: new messages
├── Text__c
├── Chat_SF_ID__c
├── Sender_Name__c
├── Sender_External_ID__c
├── Message_Type__c
├── Media_URL__c
├── Sent_At__c
└── Protocol__c

tgint__Session_Status__e       -- Go -> Salesforce: session lifecycle
├── Connection_SF_ID__c
├── Status__c                  -- 'active' | 'disconnected' | 'auth_required'
└── Error_Detail__c

tgint__Message_Delivery_Status__e  -- Go -> Salesforce: outbound delivery status
├── Message_External_ID__c
├── Status__c                  -- 'DELIVERED' | 'FAILED' | 'RETRYING'
├── Error_Code__c
├── Error_Detail__c
└── Timestamp__c
```

## Managed Package Structure (2GP)

```
Managed Package: "Messenger Integration" (namespace: tgint)
├── Custom Objects:
│   ├── tgint__Messenger_Connection__c
│   ├── tgint__Messenger_Chat__c
│   ├── tgint__Messenger_Message__c
│   └── tgint__Messenger_Event__c
├── Custom Metadata Types:
│   └── tgint__Messenger_Settings__mdt (Protected — HMAC secrets, Go server URL)
├── Platform Events:
│   ├── tgint__Inbound_Message__e
│   ├── tgint__Session_Status__e
│   └── tgint__Message_Delivery_Status__e
├── Apex Classes:
│   ├── MessengerInboundAPI.cls (@RestResource — receives webhooks)
│   ├── MessengerOutboundService.cls (callouts to Go)
│   ├── MessengerController.cls (LWC backend)
│   ├── CentrifugoTokenController.cls (JWT generation)
│   └── HMACValidator.cls (shared verification)
├── Apex Triggers:
│   ├── InboundMessageTrigger.trigger (PE -> record creation)
│   ├── SessionStatusTrigger.trigger (PE -> connection status)
│   └── DeliveryStatusTrigger.trigger (PE -> delivery status update)
├── Lightning Web Components:
│   ├── messengerChat (main chat UI)
│   ├── messengerChatList (chat list)
│   ├── messengerConnectionSetup (admin config)
│   ├── messengerAuthFlow (MTProto auth with OTP)
│   └── messengerMediaViewer (R2/CDN media)
├── Permission Sets:
│   ├── Messenger_Admin (manage connections, auth)
│   └── Messenger_User (read/send messages)
├── Connected App:
│   └── Messenger_Go_Server (OAuth 2.0)
└── CSP Trusted Sites:
    ├── Centrifugo WebSocket domain
    └── Cloudflare Worker CDN domain
```
