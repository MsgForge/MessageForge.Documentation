# Architecture Reference

## System Overview

A Go middleware server bridges messaging platforms (Telegram, WhatsApp, Viber, SMS, Apple Messages) and Salesforce. Each component has a specific role:

| Component | Role |
|---|---|
| Go Middleware | Multi-platform protocol handler (MTProto, Bot API, Cloud API, REST, MSP), message routing, media processing, queue management |
| PostgreSQL | Session storage, message queues, connection registry, rate limit state |
| Salesforce | CRM data store, user interface (LWC), business logic (Apex), event bus, credential storage (encrypted), media storage (ContentVersion) |

## Data Flow

```
                    ┌──────────────┐
                    │  Messaging   │
                    │  Platforms   │
                    │  (Telegram,  │
                    │  WhatsApp,   │
                    │  Viber, SMS, │
                    │  Apple)      │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐         ┌──────────────┐
                    │     Go       │◄───────►│  PostgreSQL   │
                    │  Middleware  │         │  (sessions,   │
                    └──────┬──────┘         │   queues)     │
                           │               └──────────────┘
                           │
                    ┌──────▼──────┐
                    │ Salesforce  │
                    │ (REST/PE)   │
                    └──┬───┬───┬──┘
                       │   │   │
          ┌────────────┘   │   └────────────┐
          │                │                │
   ┌──────▼──────┐  ┌─────▼──────┐  ┌──────▼─────────┐
   │   Apex      │  │ Platform   │  │ ContentVersion │
   │  Triggers   │  │ Events     │  │ (media files)  │
   └──────┬──────┘  └─────┬──────┘  └──────┬─────────┘
          │               │ (empApi)        │
          └───────┐  ┌────┘         ┌──────┘
                  ▼  ▼              ▼
              ┌─────────────────────┐
              │      LWC            │
              │   (Browser)         │
              └─────────────────────┘
```

## What Lives Where

### Go Side (PostgreSQL)

| Data | Table | Notes |
|---|---|---|
| MTProto sessions | `mtproto_sessions` | UNLOGGED for performance. Auth keys, salts, sequence numbers |
| Bot API config | `bot_connections` | Webhook URLs, runtime state |
| WhatsApp webhooks | `whatsapp_webhooks` | Webhook registrations, verify tokens |
| Inbound queue | `inbound_queue` | Messages awaiting Salesforce delivery |
| Outbound queue | `outbound_queue` | Messages awaiting platform sending |
| Media references | _(Salesforce ContentVersion)_ | Media metadata stored in SF, not PostgreSQL (ADR-20) |
| Rate limit state | `rate_limit_buckets` | Per-account, per-bot token buckets |
| Connection registry | `connections` | All active connections, mapped to SF Channel ID |
| Proxy assignments | `proxy_pool` | Residential proxy IPs per MTProto session |

**Important:** Go does NOT persist credentials. It reads them from Salesforce at startup (via Config + App objects), establishes connections, then holds only runtime session data.

### Salesforce Side

| Data | Custom Object | Notes |
|---|---|---|
| Channel config | `Messenger_Channel__c` | Base object — platform-agnostic status, session state |
| Platform credentials | `*_Config__c` (8 types) | Master-Detail to Channel. Encrypted secrets. |
| Platform accounts | `*_App__c` (5 types) | Account-level credentials shared across channels |
| Agent access | `Channel_User_Access__c` | Polymorphic: User or Group with access level |
| Chats | `Messenger_Chat__c` | Thread metadata, Lead/Contact link, assigned agent |
| Messages | `Messenger_Message__c` | Text, sender, media URL, direction, delivery status |
| Attachments | `Messenger_Attachment__c` | Multi-media support (albums, documents). MD → Message |
| Templates | `Messenger_Template__c` | Reusable message templates (WhatsApp required). Lookup → Channel |
| Audit trail | `Channel_Audit_Log__c` | Security events, credential rotations, session changes |
| Channel types | `Channel_Type__mdt` | Package-deployed reference (read-only for customer) |
| AES key | `Encryption_Key__mdt` | Protected CMT — invisible to customer |
| Apple MSP creds | `Apple_MSP__mdt` | Protected CMT — our Apple credentials |

### Media Storage (Salesforce ContentVersion)

All media files are stored as Salesforce ContentVersion records. Files are linked to Message/Attachment records via ContentDocumentLink. Accessed in LWC via native `/sfc/servlet.shepherd/` URLs with session-based authentication (ADR-20).

## Multi-Platform Support

### Architecture: Base + Config + App

The data model uses a three-layer pattern:

1. **Messenger_Channel__c** — platform-agnostic base. Status, session, owner, display name. No credentials.
2. **Platform Config objects** — 1:1 Master-Detail to Channel. Platform-specific credentials and settings.
3. **Platform App objects** — account-level credentials shared across multiple channels. Config → App via Lookup.

```
Channel (base, agnostic)
  └── Config (1:1 MD, platform-specific credentials)
        └── App (Lookup, shared account-level credentials)
```

This separation means:
- Channel can be queried/displayed without loading credentials
- Deleting a channel cascades to config but not to app
- App can be reused across multiple channels (e.g., one Twilio Account → many phone numbers)

### Telegram: Dual Protocol

Telegram is the only platform with a fundamental Bot vs User distinction (different protocols, different credentials, different capabilities):

| Use Case | Protocol | Config Object | Reason |
|---|---|---|---|
| Parsing public channels | MTProto | `Telegram_User_Config__c` | Bot API can't read unless bot is admin |
| Monitoring private groups | MTProto | `Telegram_User_Config__c` | Userbot must be a member |
| Sending notifications | Bot API | `Telegram_Bot_Config__c` | Cleaner UX, bot-specific features |
| Interactive menus/commands | Bot API | `Telegram_Bot_Config__c` | Inline buttons, command handlers |
| Downloading large media | MTProto | `Telegram_User_Config__c` | Bot API caps at 20MB |
| Bulk message export | MTProto | `Telegram_User_Config__c` | Full history access |

### Capability Comparison (Telegram)

| Capability | MTProto (Userbot) | Bot API |
|---|---|---|
| Receive messages | All chats, groups, channels | Only where bot is member |
| Send messages | As real person | With BOT badge |
| File download | Up to 2 GB | Up to 20 MB |
| File upload | Up to 2 GB | Up to 50 MB |
| Account type | Phone number required | Bot token from @BotFather |
| Rate limit scope | Per API ID + IP + behavior | Per bot token |

## Infrastructure

### Geographic Placement

Telegram DCs (DC2, DC4) operate from **Amsterdam, Netherlands** (AS62041). Host Go middleware near Amsterdam for minimal latency.

**Recommended providers:**
- **Hetzner** — Falkenstein/Nuremberg. Ampere ARM instances. Strong EU peering. Cost-effective
- **DigitalOcean** — Amsterdam (AMS3). Clean IP reputation
- **AWS Graviton** — `eu-west-1` (Ireland) or `eu-central-1` (Frankfurt). ARM-based

### IP Reputation

Clean, dedicated IPs required for MTProto. Budget VPS with spam-tainted subnets increase ban risk. Deploy residential/ISP proxy pools for per-session MTProto IP isolation.

### ARM & CI/CD

Go cross-compilation: `GOOS=linux GOARCH=arm64` (production) + `GOOS=linux GOARCH=amd64` (fallback).

GitHub Actions: native ARM runners or Docker Buildx (avoid QEMU). Produce single multi-arch OCI container manifest.
