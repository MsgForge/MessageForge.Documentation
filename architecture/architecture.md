# Architecture Reference

## System Overview

A Go middleware server bridges Telegram and Salesforce. Each component has a specific role:

| Component | Role |
|---|---|
| Go Middleware | Telegram protocol handler, message routing, media processing, queue management |
| PostgreSQL | Session storage, message queues, connection registry, media refs, rate limit state |
| Salesforce | CRM data store, user interface (LWC), business logic (Apex), event bus |
| Cloudflare R2 | Media file storage (images, video, voice, documents) |
| Cloudflare Workers | Edge reverse proxy for secure, cached media delivery |
| Centrifugo | Real-time WebSocket server for live chat UI updates |

## Data Flow

```
                    ┌─────────────┐
                    │   Telegram   │
                    │  MTProto/Bot │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐         ┌──────────────┐
                    │     Go       │◄───────►│  PostgreSQL   │
                    │  Middleware  │         │  (sessions,   │
                    └──┬───┬───┬──┘         │   queues)     │
                       │   │   │            └──────────────┘
              ┌────────┘   │   └────────┐
              │            │            │
      ┌───────▼──┐  ┌──────▼─────┐  ┌──▼──────────┐
      │Cloudflare│  │ Salesforce  │  │ Centrifugo   │
      │   R2     │  │ (REST/gRPC) │  │ (WebSocket)  │
      │ (media)  │  └──────┬─────┘  └──────┬───────┘
      └────┬─────┘         │               │
           │        ┌──────▼─────┐         │
           │        │   Apex     │         │
           │        │  Triggers  │         │
           │        └──────┬─────┘         │
           │               │               │
      ┌────▼───┐    ┌──────▼─────┐         │
      │Workers │    │    LWC     │◄────────┘
      │ (CDN)  │───►│ (Browser)  │
      └────────┘    └────────────┘
```

## What Lives Where

### Go Side (PostgreSQL)

| Data | Table | Notes |
|---|---|---|
| MTProto sessions | `mtproto_sessions` | UNLOGGED for performance. Auth keys, salts, sequence numbers |
| Bot API config | `bot_connections` | Encrypted tokens, webhook URLs |
| Inbound queue | `inbound_queue` | Messages awaiting Salesforce delivery |
| Outbound queue | `outbound_queue` | Messages awaiting Telegram sending |
| Media references | `media_files` | R2 object keys, Telegram file IDs, MIME types |
| Rate limit state | `rate_limit_buckets` | Per-account, per-bot token buckets |
| Connection registry | `connections` | All active connections, mapped to SF org + user |
| Proxy assignments | `proxy_pool` | Residential proxy IPs per MTProto session |

### Salesforce Side

| Data | Custom Object | Notes |
|---|---|---|
| Connection config | `Messenger_Connection__c` | Status, platform, account mapping |
| Chats | `Messenger_Chat__c` | Chat ID, title, type, last message timestamp |
| Messages | `Messenger_Message__c` | Text, sender, media URL, direction, delivery status |
| Parsed events | `Messenger_Event__c` | Extracted activities/meetups from messages |

### Cloudflare R2

All media files. Never stored in Salesforce. Zero egress fees.

## Dual Protocol: MTProto vs Bot API

Both protocols work bidirectionally — send and receive through either channel.

### When to Use Which

| Use Case | Protocol | Reason |
|---|---|---|
| Parsing public channels | MTProto | Bot API can't read unless bot is admin |
| Monitoring private groups | MTProto | Userbot must be a member |
| Sending notifications | Bot API | Cleaner UX, bot-specific features |
| Interactive menus/commands | Bot API | Inline buttons, command handlers |
| Downloading large media (>20MB) | MTProto | Bot API caps at 20MB |
| Responding to user DMs | Bot API | Users expect bot, not userbot |
| Bulk message export/archive | MTProto | Full history access, higher limits |

### Capability Comparison

| Capability | MTProto (Userbot) | Bot API |
|---|---|---|
| Receive messages | All chats, groups, channels | Only where bot is member |
| Send messages | As real person | With BOT badge |
| File download | Up to 2 GB | Up to 20 MB |
| File upload | Up to 2 GB | Up to 50 MB |
| Account type | Phone number required | Bot token from @BotFather |
| Rate limit scope | Per API ID + IP + behavior | Per bot token (IP abstracted) |
| Event fidelity | All state changes | Filtered subset |

## Infrastructure

### Geographic Placement

Telegram DCs (DC2, DC4) operate from **Amsterdam, Netherlands** (AS62041). Host Go middleware near Amsterdam for minimal latency.

**Recommended providers:**
- **Hetzner** — Falkenstein/Nuremberg. Ampere ARM instances. Strong EU peering. Cost-effective
- **DigitalOcean** — Amsterdam (AMS3). Clean IP reputation
- **AWS Graviton** — `eu-west-1` (Ireland) or `eu-central-1` (Frankfurt). ARM-based

### IP Reputation

Clean, dedicated IPs required. Budget VPS with spam-tainted subnets increase ban risk. Deploy residential/ISP proxy pools for per-session MTProto IP isolation.

### ARM & CI/CD

Go cross-compilation: `GOOS=linux GOARCH=arm64` (production) + `GOOS=linux GOARCH=amd64` (fallback).

GitHub Actions: native ARM runners or Docker Buildx (avoid QEMU). Produce single multi-arch OCI container manifest.
