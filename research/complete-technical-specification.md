# Messenger-to-Salesforce Integration: Complete Architecture & Technical Reference

> **Purpose of this document:** This is a consolidated technical specification for building an AppExchange-compliant Salesforce plugin that integrates with messaging platforms (starting with Telegram) via a Go middleware. It is designed to be consumed by AI assistants (Claude, Gemini, GPT, etc.) as a single-source-of-truth context document for development, code generation, and architectural decision-making.
>
> **Important context:** The primary developer is a Go expert with limited Salesforce experience. Salesforce concepts are explained in detail with Go/backend analogies throughout this document.

---

## Table of Contents

1. [Project Summary & Scope](#1-project-summary--scope)
2. [Technical Stack](#2-technical-stack)
3. [Salesforce Platform — Explained for Go Developers](#3-salesforce-platform--explained-for-go-developers)
4. [Telegram Protocol Layer — Dual Mode (MTProto + Bot API)](#4-telegram-protocol-layer--dual-mode-mtproto--bot-api)
5. [Rate Limiting & Concurrency in Go](#5-rate-limiting--concurrency-in-go)
6. [Authorization & Session Management](#6-authorization--session-management)
7. [Salesforce AppExchange Compliance & Security](#7-salesforce-appexchange-compliance--security)
8. [Data Ingestion Strategy](#8-data-ingestion-strategy)
9. [Real-Time UI Communication](#9-real-time-ui-communication)
10. [Media Storage & CDN Architecture](#10-media-storage--cdn-architecture)
11. [Multi-Messenger Abstraction Layer](#11-multi-messenger-abstraction-layer)
12. [Infrastructure & Hosting](#12-infrastructure--hosting)
13. [Database Architecture (Remote PostgreSQL)](#13-database-architecture-remote-postgresql)
14. [Fallback & Queue Design](#14-fallback--queue-design)
15. [Security Checklist](#15-security-checklist)
16. [Key Architectural Decisions Log](#16-key-architectural-decisions-log)
17. [Glossary](#17-glossary)

---

## 1. Project Summary & Scope

### 1.1 What We Are Building

A Salesforce AppExchange plugin (Managed Package, classified as a "Composite App") that connects Salesforce CRM to messaging platforms. A Go-based middleware server acts as the bridge.

### 1.2 MVP Scope (Telegram Only)

The MVP focuses exclusively on Telegram with **two protocol modes, both fully supported:**

- **MTProto (Userbots):** For parsing public/private channels and groups with full-fidelity event streams AND for sending messages as a user account. Requires phone-number-based authorization managed by a Salesforce Administrator using corporate SIM cards.
- **Bot API:** For user-facing bot interactions (menus, commands, inline keyboards, notifications) AND for receiving webhook updates. Uses standard bot tokens.

**Both protocols work bidirectionally** — the system can both receive and send messages through either channel. A Salesforce user can view a chat timeline that mixes userbot-parsed messages with bot-delivered messages, and can respond through either pathway depending on the use case.

### 1.3 Primary Use Case

A local activities and events aggregator — parsing regional Telegram channels and chats (initially across Cyprus) to extract event data, storing structured results in Salesforce, and providing a chat interface within the Salesforce Lightning UI.

### 1.4 Future Scope (Post-MVP)

The architecture must be designed to support additional messenger integrations in the future:
- WhatsApp Business API
- Facebook Messenger
- Instagram Direct
- Viber
- Custom webhook-based messengers

This means the Go middleware must implement a **messenger-agnostic abstraction layer** (see Section 11) so that adding a new messenger is a matter of writing a new adapter, not restructuring the entire system.

---

## 2. Technical Stack

| Component | Technology | Notes |
|---|---|---|
| **Middleware** | Go (Golang) | Compiled for ARM (arm64). High concurrency via goroutines. |
| **MTProto Engine** | `gotd/td` | Low-level, pure Go MTProto 2.0 client. ~150KB memory per idle client. Actively maintained (last commit Feb 2026). 2,125+ GitHub stars. |
| **MTProto Wrapper** | `GoTGProto` (`celestix/gotgproto`) | Simplifies session management, peer storage, handler routing on top of `gotd/td`. Beta stage. |
| **CRM Platform** | Salesforce | Apex (server-side logic), Lightning Web Components (LWC, frontend UI), SOQL (database queries). |
| **Database** | PostgreSQL (remote, hosted) | Dedicated PostgreSQL instance on the Go server side. Stores sessions, message queues, metadata, media references. NOT local/embedded — a proper hosted DB. |
| **Object Storage** | Cloudflare R2 | S3-compatible, zero egress fees. Used for media files (images, video, voice notes). |
| **CDN / Edge Auth** | Cloudflare Workers | Reverse proxy for secure media delivery with HMAC-signed URLs. |
| **Real-time Broker** | Centrifugo | Go-based WebSocket server for pushing live chat updates to Salesforce LWC, bypassing Salesforce event limits. |
| **CI/CD** | GitHub Actions | Multi-arch builds (amd64 + arm64) using native runners or Docker Buildx. |

---

## 3. Salesforce Platform — Explained for Go Developers

This section explains Salesforce concepts using Go/backend analogies. If you are a Go developer and have never worked with Salesforce, read this section carefully.

### 3.1 What Salesforce Actually Is

Think of Salesforce as a **hosted, multi-tenant application platform with a built-in database, ORM, REST API, UI framework, and deployment pipeline** — all in one. You don't provision servers. You don't manage a database. Everything runs on Salesforce's infrastructure.

**Go analogy:** Imagine if your Go app, its PostgreSQL database, its REST API, and its React frontend were all hosted on a single platform that enforced strict resource quotas per tenant and auto-scaled for you. That's Salesforce.

### 3.2 Key Concepts Map

| Salesforce Term | Go/Backend Equivalent | Explanation |
|---|---|---|
| **Org** | A tenant / deployment environment | One Salesforce "organization" = one customer's entire instance. Each org has its own data, code, and configuration. |
| **Object** | A database table | `Account`, `Contact`, `Case` are standard objects (built-in tables). You create **Custom Objects** like `Telegram_Message__c` (the `__c` suffix means "custom"). |
| **Field** | A column in a table | Each object has fields. Custom fields also get the `__c` suffix: `Message_Text__c`. |
| **Record** | A row in a table | One instance of an object. e.g., one specific Telegram message stored in the `Telegram_Message__c` object. |
| **Apex** | Server-side Java-like code (runs on Salesforce) | Salesforce's proprietary language. Syntax is almost identical to Java. It runs ON Salesforce servers, not yours. You use it for business logic, triggers, REST endpoints. |
| **SOQL** | SQL (but simpler, with restrictions) | Salesforce Object Query Language. Example: `SELECT Id, Name FROM Account WHERE Name = 'Acme'`. No JOINs — uses relationship traversal instead. |
| **Trigger** | A database trigger / event hook | Apex code that runs automatically before or after a record is inserted, updated, or deleted. Like PostgreSQL triggers but written in Apex. |
| **Lightning Web Components (LWC)** | React/Vue components | JavaScript-based UI framework. Components use HTML templates + JS controllers. They run in the user's browser inside the Salesforce UI. |
| **Managed Package** | A distributable app/plugin (like a Go module, but for an entire app) | A bundle of custom objects, Apex classes, LWC components, etc., packaged together and distributed via AppExchange (Salesforce's app store). |
| **Namespace** | A Go module path | Managed packages get a unique namespace prefix (e.g., `tgint__`) that prefixes all their objects and fields to prevent naming conflicts. So your custom object becomes `tgint__Telegram_Message__c`. |
| **Governor Limits** | Resource quotas (like ulimits, but enforced per transaction) | Hard runtime limits enforced by Salesforce because it's multi-tenant. If your code exceeds a limit, it throws an unhandleable exception and the transaction rolls back. |
| **Platform Events** | A pub/sub message bus (like NATS, Kafka topics) | Events published to Salesforce's internal event bus. External systems can publish events IN, and LWC components can subscribe to them in real-time. |
| **Flow** | A visual automation builder (no-code) | Drag-and-drop workflow automation. Can trigger on record changes, scheduled times, or platform events. Admins (non-developers) build these. |
| **Connected App** | An OAuth2 client registration | Defines how an external application (your Go middleware) authenticates to Salesforce APIs. Contains client ID, client secret, scopes, callback URLs. |
| **Protected Custom Metadata Type** | An encrypted config store (like Vault secrets) | Key-value configuration stored inside Salesforce, accessible only within the managed package. Used for storing shared secrets (like HMAC keys). Cannot be read by org admins. |

### 3.3 Governor Limits — The Critical Ones

Governor limits are the most important thing for a Go developer to understand about Salesforce. They will fundamentally shape every design decision.

**Per Apex Transaction (every single request):**

| Limit | Synchronous | Asynchronous |
|---|---|---|
| SOQL queries | 100 | 200 |
| DML statements (insert/update/delete) | 150 | 150 |
| Records retrieved by SOQL | 50,000 | 50,000 |
| Heap size | 6 MB | 12 MB |
| CPU time | 10,000 ms | 60,000 ms |
| Callouts (HTTP requests to external services) | 100 | 100 |
| Callout timeout | 120 seconds | 120 seconds |

**Platform-Wide Limits (per org, per 24 hours):**

| Limit | Value |
|---|---|
| API requests (REST/SOAP) | Varies by edition + number of licenses (e.g., Enterprise: 1,000 per user license per 24h, minimum 15,000) |
| Platform Events published | Up to 250,000/hour (Performance/Unlimited) |
| Platform Events delivered (external) | 50,000/24h default — **the real bottleneck**. Each event × each subscriber = one delivery. |
| Bulk API batches | 15,000/24h rolling |
| Data storage | ~10 MB per user license |
| File storage | ~2 GB per user license |

**Why this matters for your Go middleware:** Every REST API call your Go server makes to Salesforce counts against the org's 24-hour API limit. If you make one API call per incoming Telegram message, a busy chat group will exhaust the limit in hours. This is why batching, Platform Events, and the Bulk API exist.

### 3.4 The Salesforce Data Model We Need to Build

These are the Custom Objects (database tables) that our Managed Package will create inside the customer's Salesforce org:

```
Messenger_Connection__c        -- A configured messenger account (Telegram phone, bot token, etc.)
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
├── Delivery_Status__c         -- 'pending' | 'delivered' | 'failed' (for outbound tracking)
└── Protocol__c                -- 'mtproto' | 'bot_api'

Messenger_Event__c             -- Parsed events (activities, meetups, etc.)
├── Message__c                 -- Lookup to source Messenger_Message__c
├── Event_Title__c
├── Event_Date__c
├── Event_Location__c
├── Event_Description__c
└── Processing_Status__c       -- 'pending' | 'confirmed' | 'discarded'
```

### 3.5 Apex Code We Will Need to Write

For a Go developer, think of Apex as "Java that runs inside Salesforce." Here's what we need:

**1. REST Endpoint (receives webhooks from Go middleware):**
```apex
// This is like a Go HTTP handler, but running inside Salesforce
@RestResource(urlMapping='/api/messenger/inbound')
global class MessengerInboundAPI {
    
    @HttpPost
    global static String handleInbound() {
        // Get the raw request body (like reading r.Body in Go)
        RestRequest req = RestContext.request;
        String body = req.requestBody.toString();
        
        // Verify HMAC signature (like middleware auth in Go)
        String signature = req.headers.get('X-Signature');
        String secret = Messenger_Settings__mdt.getInstance('HMAC_Secret').Value__c;
        
        Blob mac = Crypto.generateMac('HmacSHA256', Blob.valueOf(body), Blob.valueOf(secret));
        String expectedSig = EncodingUtil.convertToHex(mac);
        
        if (signature != expectedSig) {
            RestContext.response.statusCode = 401;
            return 'Invalid signature';
        }
        
        // Parse JSON and create records
        // ... (message processing logic)
        return 'OK';
    }
}
```

**2. Platform Event Subscriber (Apex Trigger):**
```apex
// This fires automatically when a Platform Event arrives
// Think of it like a Go channel receiver
trigger InboundMessageTrigger on Inbound_Message__e (after insert) {
    List<Messenger_Message__c> toInsert = new List<Messenger_Message__c>();
    
    for (Inbound_Message__e event : Trigger.New) {
        toInsert.add(new Messenger_Message__c(
            Message_Text__c = event.Text__c,
            Chat__c = event.Chat_SF_ID__c,
            Direction__c = 'inbound'
        ));
    }
    
    insert toInsert; // Bulk insert — one DML for all records
}
```

**3. LWC Component (chat UI in user's browser):**
```javascript
// This is a Lightning Web Component — basically a React component
// File: messengerChat.js
import { LightningElement, wire, track } from 'lwc';
import { subscribe } from 'lightning/empApi';  // Subscribes to Platform Events
import getMessages from '@salesforce/apex/MessengerController.getMessages';

export default class MessengerChat extends LightningElement {
    @track messages = [];
    chatId;
    
    // This is like useEffect + fetch in React
    @wire(getMessages, { chatId: '$chatId' })
    wiredMessages({ data, error }) {
        if (data) this.messages = data;
    }
    
    // Subscribe to real-time Platform Events (like a WebSocket listener)
    connectedCallback() {
        subscribe('/event/Inbound_Message__e', -1, (message) => {
            // New message arrived — update the UI
            this.messages = [...this.messages, message.data.payload];
        });
    }
}
```

### 3.6 How Salesforce Authenticates Your Go Middleware

Your Go server needs to call Salesforce APIs. There are two main flows:

**OAuth 2.0 JWT Bearer Flow (recommended for server-to-server):**
1. Create a **Connected App** in Salesforce Setup.
2. Generate an X.509 certificate and upload it to the Connected App.
3. Your Go middleware creates a JWT signed with the private key.
4. Exchange the JWT for an access token via `POST /services/oauth2/token`.
5. Use the access token in the `Authorization: Bearer <token>` header for all API calls.
6. Tokens expire — your Go server must handle refresh.

**Go side (your middleware) calls Salesforce like this:**
```
POST https://<instance>.salesforce.com/services/data/v60.0/sobjects/Messenger_Message__c
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "Message_Text__c": "Hello from Telegram",
  "Chat__c": "a0B5f000001234567",
  "Direction__c": "inbound"
}
```

### 3.7 Salesforce Editions & What They Mean for Us

| Edition | API Calls/24h | Platform Events | File Storage | Price |
|---|---|---|---|---|
| **Essentials** | 15,000 | Limited | 1 GB | ~$25/user/mo |
| **Professional** | 15,000 | Limited | 10 GB | ~$80/user/mo |
| **Enterprise** | 100,000+ | 250K/hr publish | 10 GB | ~$165/user/mo |
| **Unlimited** | Unlimited | 250K/hr publish | Unlimited | ~$330/user/mo |

**Our target:** Enterprise edition minimum. Professional edition will be too constrained for active Telegram parsing.

### 3.8 Development Tools & Environments

| Tool | What It Does | Go Equivalent |
|---|---|---|
| **Salesforce CLI (sf)** | Deploy code, create scratch orgs, run tests | `go build` + `go test` + deployment tool |
| **VS Code + Salesforce Extensions** | IDE for Apex + LWC development | VS Code + Go extension |
| **Scratch Org** | Disposable, temporary Salesforce environment for development | Docker container for testing |
| **Sandbox** | A copy of a production org for staging/QA | Staging environment |
| **Developer Edition Org** | Free Salesforce org for development (limited features) | Free tier cloud account |
| **Package Version** | A snapshot of your managed package code, installable by customers | A tagged release / Docker image |

---

## 4. Telegram Protocol Layer — Dual Mode (MTProto + Bot API)

### 4.1 Both Protocols Fully Supported, Bidirectionally

The system supports **both** MTProto and Bot API as first-class citizens. Both can send and receive:

| Capability | MTProto (Userbot) | Bot API |
|---|---|---|
| **Receive messages** | Full-fidelity: all chats, groups, channels, including private ones the user is a member of | Webhook or long-poll: only messages directed at the bot or in groups where bot is a member |
| **Send messages** | As the user account (appears as a real person) | As the bot (appears with "BOT" badge) |
| **Parse channels** | Can read any public channel + channels user has joined | Only channels where bot is an admin |
| **File download** | Up to 2 GB | Up to 20 MB |
| **File upload** | Up to 2 GB | Up to 50 MB |
| **Account type** | Real phone number required | Bot token from @BotFather |
| **Rate limit scope** | Per API ID + IP + account behavior | Per bot token (IP abstracted) |
| **Event fidelity** | All state changes (typing, read receipts, edits, deletes, pins, reactions) | Filtered subset (messages, edits, some callbacks) |
| **User data access** | Full user profiles, phone numbers (if shared), online status | Limited: first name, last name, username, user ID |

### 4.2 When to Use Which Protocol

| Use Case | Protocol | Reason |
|---|---|---|
| Parsing public channels for events | MTProto | Bot API can't read channels unless bot is admin |
| Monitoring private group conversations | MTProto | Userbot must be a member |
| Sending notifications to users | Bot API | Cleaner UX, bot-specific features (keyboards, callbacks) |
| Interactive menus / commands | Bot API | Designed for this — inline buttons, command handlers |
| Downloading large media (>20MB) | MTProto | Bot API caps at 20MB download |
| Responding to user DMs | Bot API | Users expect to talk to a bot, not a userbot |
| Bulk message export / archive | MTProto | Full history access, higher limits |

### 4.3 MTProto — Capability and Risk Details

**Capabilities:** Full `gotd/td` MTProto 2.0 implementation. Direct TCP connection to Telegram data centers. Access to all TL schema methods.

**Multi-Tenancy Risks:**
- `api_id` + `api_hash` represent the developer application, not the user session.
- Shared API ID across unrelated users from a single IP triggers `API_ID_PUBLISHED_FLOOD` or permanent bans.
- Each phone number can only have one `api_id` associated with it.
- Automating API ID creation via `my.telegram.org` is a severe ToS violation leading to account termination and subnet blacklisting.

**Mitigation:** Single corporate `api_id` + dedicated residential/ISP proxy IPs per user session.

### 4.4 Bot API — Capability Details

**Capabilities:** Standard HTTPS webhook receiver. Go middleware registers a webhook URL with Telegram via `setWebhook`. Telegram pushes JSON updates to that URL.

**Rate limits:**
- ~30 messages/second globally for mass mailing
- ~1 message/second per individual DM
- ~20 messages/minute per group/channel

### 4.5 Rate Limits — Full Reference

**MTProto (Userbots):**
- ~50 cold DMs per day maximum for new/untrusted accounts
- ~1 operation per second per chat for automated interactions
- `FLOOD_WAIT` errors include an explicit wait duration (seconds) in the error payload
- Bursting without historic account trust triggers penalties

**Bot API:**
- ~30 messages/second globally (mass mailing via `sendMessage`)
- ~1 message/second per individual DM
- ~20 messages/minute per group/channel
- Rate limits scoped to bot token, not server IP

---

## 5. Rate Limiting & Concurrency in Go

### 5.1 Middleware Chain (gotd, for MTProto)

The `gotd/td` library uses composable middleware wrapping MTProto RPC invocations:

1. **`ratelimit` middleware** (proactive): Local token-bucket throttle. Example: cap at 5 operations per 100ms. Smooths bursty traffic before it hits Telegram servers.
2. **`floodwait` middleware** (reactive): `floodwait.NewWaiter()` wraps the client runner. On `FLOOD_WAIT_X`, suspends the operation for X seconds and retries — without dropping the TCP connection or invalidating the cryptographic handshake.

**Order:** `ratelimit` → `floodwait` → actual RPC call.

### 5.2 Bot API Throttling (separate from MTProto)

For Bot API, implement a per-bot token bucket in Go:
- Global bucket: 30 tokens/second (mass send)
- Per-chat bucket: 1 token/second (DMs), ~1 token/3 seconds (groups)
- Queue outgoing messages in PostgreSQL; worker goroutines consume from the queue respecting rate limits.

### 5.3 Concurrency with `errgroup`

Replace raw goroutines with `errgroup` from `golang.org/x/sync`:
- Bounded worker pools with strict error propagation
- Context cancellation on failure (graceful degradation, not zombie goroutines)
- Critical for MTProto handlers recovering `pts`/`qts` sequence numbers after network interruption

### 5.4 Distributed Throttling (Multi-Node)

For horizontally scaled deployments, use Redis as the global rate-limit state store:
- Token Bucket / Sliding Window via atomic Redis operations + Lua scripts
- Namespace keys by `tenant_id` (Salesforce org ID)
- Prevents one org's traffic surge from exhausting another's allocation

---

## 6. Authorization & Session Management

### 6.1 MTProto: Admin-Driven Authorization Flow

1. Admin opens a custom LWC component in Salesforce.
2. LWC connects to Centrifugo (external WebSocket server) — domain must be in Salesforce CSP Trusted Sites.
3. Admin enters phone number; Go middleware initiates MTProto `auth.sendCode`.
4. Telegram sends OTP to the phone. Admin enters it in the LWC.
5. OTP relayed in real-time via WebSocket to Go middleware.
6. Go middleware executes `auth.signIn`. If 2FA is NOT enabled, session is established immediately.
7. **If 2FA IS enabled:** Telegram responds with `SESSION_PASSWORD_NEEDED` RPC error (400). This is NOT a failure — it is a mandatory transitional state requiring SRP proof (see 6.1.1 below).
8. Session established. Cryptographic keys stored in PostgreSQL (mapped to Salesforce User ID).

**Why WebSocket, not long-polling?** OTP codes expire quickly. WebSocket provides sub-second latency needed before timeout.

#### 6.1.1 SRP v6a: Two-Factor Authentication (Cloud Password)

When `SESSION_PASSWORD_NEEDED` is intercepted, the Go middleware must execute the Secure Remote Password protocol v6a. The password is **never transmitted in plaintext** — the middleware proves knowledge of the password via a zero-knowledge proof.

**Step-by-step cryptographic flow:**

1. Go middleware calls `account.getPassword` → Telegram returns SRP parameters:
   - KDF algorithm: `passwordKdfAlgoSHA256SHA256PBKDF2HMACSHA512iter100000SHA256ModPow`
   - Two salt values: `salt1`, `salt2`
   - Safe 2048-bit prime `p`, generator `g`, server public key `g_b`

2. **Primary hash:** Password concatenated with salt values, subjected to nested SHA-256 hashing:
   `H = SHA256(salt2 || SHA256(salt1 || password || salt1) || salt2)`

3. **PBKDF2:** Apply PBKDF2 with HMAC-SHA512 over **exactly 100,000 iterations**. This is CPU-intensive by design (brute-force protection). Must run on the Go backend — attempting this in LWC JavaScript would lock the browser UI.

4. **Secondary hash:** Output of PBKDF2 hashed again with salt2 via SHA-256.

5. **Client key generation:** Extract scalar value `x` from the hash, generate a cryptographically secure random 2048-bit number `a`, compute client public key `g_a = g^a mod p`.

6. **Session key derivation:** Compute scrambling parameter `u = SHA256(g_a || g_b)`, derive shared session key `K` and verification proof `M1`.

7. Package `g_a` and `M1` into an `InputCheckPasswordSRP` object and transmit via `auth.checkPassword`.

**Library support:** The `gotd/td` library (specifically `github.com/gotd/td/internal/crypto/srp`) handles the full SRP computation. Use `telegram/auth` package which abstracts the flow.

#### 6.1.2 LWC UI State Machine for Authorization

The LWC must be a dynamic state machine that adapts its interface based on the authorization phase:

| Phase | Trigger | Go Middleware Action | LWC Renders |
|---|---|---|---|
| **1: Init** | Admin inputs phone number | Executes `auth.sendCode` | Phone number input field |
| **2: OTP** | Admin inputs Telegram OTP | Executes `auth.signIn` | OTP numeric input (phone field unmounted) |
| **3: Eval** | Middleware intercepts `SESSION_PASSWORD_NEEDED` | Calls `account.getPassword` | Loading indicator |
| **4: 2FA** | Middleware pushes 2FA request via WebSocket | Awaits password string | Secure password input field (OTP field unmounted) |
| **5: Proof** | Admin submits password | Executes PBKDF2 + `auth.checkPassword` | Progress indicator → success → active chat interface |

This explicit multi-phase state management prevents authorization deadlocks and ensures seamless UX for admins connecting new userbot instances.

### 6.2 Bot API: Token Registration Flow

1. Admin creates a bot via Telegram's @BotFather and receives a bot token.
2. Admin enters the bot token in a custom LWC settings page.
3. Token is transmitted to the Go middleware via Salesforce REST callout (or direct HTTPS to Go server).
4. Go middleware calls `setWebhook` on the Telegram Bot API to register its webhook URL.
5. Bot token is stored in PostgreSQL (encrypted), mapped to the Salesforce org.

### 6.3 Session Key Storage

**Decision: All session data stored in remote PostgreSQL on the Go server side, NOT in Salesforce.**

Rationale:
- MTProto sessions require frequent, sequential state updates (auth keys, salts, sequence numbers).
- Salesforce Protected Custom Metadata Types require async Metadata API deployment — impractical for high-frequency writes.
- Routing every `StoreSession` through Salesforce REST adds fatal latency and race conditions.

**What Salesforce stores:** Integration metadata only — connection status, Salesforce User ID ↔ Telegram account mapping, last-seen timestamps. These are stored in the `Messenger_Connection__c` custom object.

### 6.4 Session Lifecycle — Forced Termination

If a session is revoked (Telegram emits `AUTH_KEY_DUPLICATED` or `AUTH_KEY_UNREGISTERED`):

1. Go middleware intercepts the error.
2. Terminates local execution context for that session.
3. Publishes a Platform Event to Salesforce event bus.
4. Apex trigger catches the event → updates `Messenger_Connection__c.Status__c` to `'auth_required'`.
5. Admin sees the status change in Salesforce UI and re-authorizes.

---

## 7. Salesforce AppExchange Compliance & Security

### 7.1 AppExchange Onboarding — Step by Step

1. **Partner Community Enrollment:** Register in the Salesforce Partner Community.
2. **Architecture Review & Build:** Declare as "Composite App" (on-platform: Apex + LWC; off-platform: Go middleware). Prepare external components for pen testing.
3. **Listing Creation:** Populate metadata in Listing Builder. Negotiate the Partner Application Distribution Agreement (PADA).
4. **Security Review:** Submit package + test environments + scan reports for all external components to Product Security team.
5. **Publication:** Upon passing, approved for AppExchange distribution.

### 7.2 Security Review Requirements for the Go Server

- **DAST scan reports:** OWASP ZAP, Burp Suite, or Chimera. Zero high-severity vulnerabilities (XSS, SQL injection, OS command injection). False positives documented.
- **TLS 1.2+ enforcement:** SSL Labs PDF/screenshot proving all endpoints enforce TLS 1.2+.
- **No hardcoded credentials:** Use environment variables or Salesforce Protected Custom Metadata Types.
- **Transient data handling:** Prove secure handling without unauthorized logging.

### 7.3 Webhook Authentication — HMAC Signature Verification

**Go → Salesforce (inbound to Salesforce):**
1. Go computes SHA-256 HMAC of raw JSON payload using pre-shared secret.
2. Injects signature into `X-Signature` HTTP header.
3. Salesforce Apex REST endpoint retrieves shared secret from Protected Custom Metadata Type.
4. Independently computes HMAC, compares using `Crypto.verifyHMac()` (constant-time).
5. Match → authenticated. Mismatch → 401 rejected.

**Salesforce → Go (outbound from Salesforce):**
When a Salesforce user sends a message through the LWC, the Apex controller calls out to the Go middleware. The Go server validates the Salesforce access token (OAuth) or a similar HMAC mechanism on its end.

### 7.4 Managed Package Structure

Our Managed Package (2GP — second-generation managed package, recommended by Salesforce) contains:

```
Managed Package: "Messenger Integration" (namespace: tgint)
├── Custom Objects:
│   ├── tgint__Messenger_Connection__c
│   ├── tgint__Messenger_Chat__c
│   ├── tgint__Messenger_Message__c
│   └── tgint__Messenger_Event__c
├── Custom Metadata Types:
│   └── tgint__Messenger_Settings__mdt (Protected — stores HMAC secrets, Go server URL)
├── Platform Events:
│   ├── tgint__Inbound_Message__e (Go → Salesforce: new messages)
│   ├── tgint__Session_Status__e (Go → Salesforce: session lifecycle changes)
│   └── tgint__Message_Delivery_Status__e (Go → Salesforce: outbound message delivery status)
├── Apex Classes:
│   ├── MessengerInboundAPI.cls (@RestResource — receives webhooks from Go)
│   ├── MessengerOutboundService.cls (Sends messages via callout to Go server)
│   ├── MessengerController.cls (LWC backend — queries messages, chats)
│   ├── CentrifugoTokenController.cls (Generates JWTs for Centrifugo WebSocket auth)
│   └── HMACValidator.cls (Shared HMAC verification logic)
├── Apex Triggers:
│   ├── InboundMessageTrigger.trigger (Platform Event → record creation)
│   ├── SessionStatusTrigger.trigger (Platform Event → connection status update)
│   └── DeliveryStatusTrigger.trigger (Platform Event → update Delivery_Status__c on Messenger_Message__c)
├── Lightning Web Components:
│   ├── messengerChat (Main chat UI)
│   ├── messengerChatList (List of active chats)
│   ├── messengerConnectionSetup (Admin: connection configuration)
│   ├── messengerAuthFlow (Admin: MTProto authorization with OTP)
│   └── messengerMediaViewer (Displays images/video from R2/CDN)
├── Permission Sets:
│   ├── Messenger_Admin (Full access: manage connections, auth)
│   └── Messenger_User (Read/send messages, view chats)
├── Connected App:
│   └── Messenger_Go_Server (OAuth 2.0 config for Go middleware auth)
└── CSP Trusted Sites:
    ├── Centrifugo WebSocket domain
    └── Cloudflare Worker CDN domain
```

---

## 8. Data Ingestion Strategy

### 8.1 API Options for Go → Salesforce

| API | Paradigm | Best For | Volume | Governor Limit Impact |
|---|---|---|---|---|
| REST API | Synchronous | Single-record UI mutations | Low (up to 200/batch) | Depletes 24hr API request limits |
| Bulk API 2.0 | Asynchronous | Historical data syncs | High (>2,000 records) | Limits based on daily record volume |
| Platform Events | Event-Driven (Pub/Sub) | Real-time UI updates | Medium (1 MB payload cap) | Hourly event delivery allocations |
| Pub/Sub API (gRPC) | Bidirectional streaming | Publishing + subscribing from Go | High throughput | Same Platform Event allocations |

### 8.2 Recommended Hybrid Approach

- **Real-time chat messages → Platform Events** via Pub/Sub API (gRPC) from Go. LWC subscribes via `empApi`.
- **Historical chat archive sync → Bulk API 2.0** (CSV/JSON upload, auto-chunked).
- **Single-record updates → REST API** (e.g., updating a connection status).

### 8.3 Platform Events — Limits & Batching

**Limits (Performance/Unlimited edition):**
- Publishing: 250,000 events/hour
- Delivery: 50,000/24h for external subscribers (each event × each subscriber = one delivery)
- Payload: 1 MB max per event

**Batching on Go server — Dynamic Payload Batching with Sliding Windows:**

Buffer incoming Telegram messages in a 2–3 second sliding window. Aggregate into a single JSON array. Publish as ONE Platform Event. Reduces event consumption by orders of magnitude.

#### 8.3.1 Why Naive Batching Fails

A naive approach — appending messages to a slice, calling `json.Marshal()` on the entire slice, and checking `len(bytes)` — is computationally disastrous under high-throughput loads. Successive marshaling of growing slices forces continuous heap allocation of new memory blocks, triggering frequent GC sweeps, CPU throttling, and latency spikes.

#### 8.3.2 Optimized Memory Strategy: sync.Pool + Streaming Encoders

Use `sync.Pool` to maintain a pool of pre-allocated `bytes.Buffer` objects. This avoids requesting new memory from the OS for every serialization:

1. **Buffer initialization:** Define a global `sync.Pool` that provides initialized `bytes.Buffer` instances to worker goroutines.
2. **Streaming serialization:** As each MTProto message arrives, retrieve an empty buffer from the pool, instantiate `json.NewEncoder(buf)`, serialize the single message struct into the buffer.
3. **Byte tracking:** Read `buf.Len()` for the exact serialized size of that individual message. Maintain a `current_batch_size` running total.
4. **Threshold evaluation:** Before appending to the batch, check:
   `current_batch_size + single_message_size + overhead > 950 KB → flush`
   (950 KB instead of 1,000,000 bytes to account for HTTP header overhead, JSON array brackets, and commas.)
5. **Buffer return:** After reading the length, return the buffer to the pool via `pool.Put(buf)` after `buf.Reset()`.

#### 8.3.3 Sliding Window Execution

- If threshold is breached → immediately isolate the current batch slice, dispatch it to an async goroutine that POSTs to the Salesforce Platform Event endpoint. Reset counter to zero, start new batch with the oversized message as its first element.
- If the 3-second temporal window expires before threshold → dispatch the current batch regardless of size. This prevents artificial latency during low-volume periods.
- Result: the Go middleware never violates the 1 MB Salesforce limit and maintains a nearly flat memory allocation profile under intense throughput bursts.

### 8.4 Salesforce Pub/Sub API (gRPC) — For Go Middleware

The Pub/Sub API (GA since June 2022) is Salesforce's gRPC-based replacement for legacy CometD:

- ~7–10x faster than REST (compressed protobuf vs JSON)
- Bidirectional streaming over HTTP/2
- Pull-based: client controls how many events to receive
- Endpoint: `api.pubsub.salesforce.com:7443`
- Events are Avro-encoded — requires schema-based deserialization
- Go has excellent gRPC support (`google.golang.org/grpc`)

**Use from Go middleware for:** Publishing inbound messages, subscribing to outbound send requests from Salesforce.

---

## 9. Real-Time UI Communication

### 9.1 The Problem

Salesforce cannot host WebSocket connections. Platform Events have hourly limits that active chat exhausts quickly.

### 9.2 Solution — Centrifugo (External WebSocket Server)

1. Deploy Centrifugo alongside Go middleware.
2. LWC components connect to Centrifugo via JavaScript WebSocket (domain in CSP Trusted Sites).
3. Go middleware publishes real-time messages to Centrifugo channels.
4. LWC receives instant updates without hitting Salesforce event limits.

**Salesforce still updated** — asynchronously via batched REST or Platform Events for persistence and search. WebSocket handles "live" UI only.

### 9.3 Centrifugo JWT Authentication from Salesforce Apex

Centrifugo is stateless — it has no awareness of Salesforce sessions. The LWC must present a cryptographically signed JWT during the WebSocket handshake to prove authorization.

#### 9.3.1 JWT Claims Structure

| Claim | Purpose | Salesforce Source |
|---|---|---|
| `sub` (Subject) | Unique user identifier. If absent, Centrifugo treats connection as anonymous. | `UserInfo.getUserId()` or Federation ID for SSO |
| `exp` (Expiration) | UNIX timestamp for token expiry. Validated during handshake only. | Current epoch + short TTL (e.g., 15 minutes) |
| `expire_at` | Centrifugo-specific: when to forcefully terminate the WebSocket connection. | Set equal to or slightly beyond `exp` |
| `info` (optional) | Custom JSON payload injected into the token. | Omni-Channel queues, user roles, interface preferences |

#### 9.3.2 Apex JWT Generation — Symmetric (HMAC-SHA256)

For internal architectures, symmetric HMAC-SHA256 is performant. The shared secret (`hmac_secret_key` in Centrifugo config) must be stored in a Protected Custom Metadata Type, Named Credential, or Encrypted Custom Field.

**Critical Apex nuance — Base64Url encoding:** Salesforce's `EncodingUtil.base64Encode()` produces standard Base64 which uses `+`, `/`, and `=` padding. These are incompatible with the JWT specification (RFC 7515). The Apex implementation **must** perform explicit character substitution:
- Replace all `+` with `-`
- Replace all `/` with `_`
- Strip all trailing `=` padding

Failure to do this results in Centrifugo immediately rejecting the token. The signature is generated via `Crypto.generateMac('HmacSHA256', payload, secret)`.

#### 9.3.3 Apex JWT Generation — Asymmetric (RSA-SHA256)

For cross-organizational key segregation, use RSA-SHA256:
- Generate a Self-Signed Certificate in Salesforce (Certificate and Key Management, minimum 2048-bit key).
- Private key stays locked in Salesforce's multi-tenant HSM.
- Export public key in PEM format → configure in Centrifugo via `rsa_public_key` directive.
- Use `Auth.JWT` class (`.setSub()`, `.setValidityLength()`, `.setAdditionalClaims()`) and `Auth.JWS` class for automated RSA signing. This handles Base64Url encoding natively — no manual string manipulation needed.

#### 9.3.4 LWC Token Refresh Lifecycle

1. LWC calls an Apex controller in `connectedCallback()` to get the initial JWT.
2. Centrifugo SDK is initialized with a `getToken` callback function.
3. When the WebSocket connection approaches the `expire_at` timestamp, the SDK automatically invokes `getToken`.
4. The LWC makes a subsequent imperative Apex call to mint a fresh JWT.
5. Centrifugo rotates the token without full WebSocket disconnection/reconnection — no latency gap in the chat UI.

### 9.4 CSP & CORS Configuration

- **Salesforce:** Add external origin URLs to CORS Allowlist in Setup.
- **Centrifugo / Cloudflare Worker:** Handle `OPTIONS` preflight; return `Access-Control-Allow-Origin` and `Access-Control-Allow-Methods` headers.
- Both sides must be configured — missing either causes CORS errors.

---

## 10. Media Storage & CDN Architecture

### 10.1 Architecture — Cloudflare R2 + Workers (Inbound Media)

```
Telegram → Go Middleware → Cloudflare R2 (private bucket, zero egress)
                                ↓
                        Cloudflare Worker (edge reverse proxy, HMAC validation)
                                ↓
                    Salesforce LWC (renders media via <img>/<video>)
```

**Incoming media:** Go intercepts → uploads to R2 → stores R2 object key + metadata in PostgreSQL → stores media URL in Salesforce `Messenger_Message__c.Media_URL__c`.

**Displaying media:** LWC requests from custom CDN domain → Worker validates HMAC-signed URL → fetches from private R2 → serves with edge caching.

### 10.2 Outbound Media Upload — Direct-to-R2 via Presigned URLs

When a Salesforce agent sends media (images, video, PDFs) to a Telegram user, the file must NOT be routed through Salesforce storage (2 GB per-user quota, database bloat, API concurrency degradation). Instead, the LWC uploads directly to Cloudflare R2 via presigned URLs, bypassing Salesforce entirely.

#### 10.2.1 Presigned URL Generation Pipeline

1. User clicks "Attach File" in the LWC chat UI.
2. LWC extracts file metadata only (filename, byte size, MIME type) — does NOT read the file into memory.
3. LWC sends metadata as JSON to Go middleware (via Centrifugo WebSocket or dedicated REST endpoint).
4. Go middleware generates a presigned PUT URL using `aws-sdk-go-v2`:
   - Instantiates `s3.PresignClient` → calls `PresignPutObject`
   - URL encodes: target bucket, UUID object key, HTTP PUT verb, expiration (15–60 minutes)
   - SigV4 signature embedded in query string (`X-Amz-Algorithm=AWS4-HMAC-SHA256`)
5. Go middleware returns the presigned URL to the LWC.

#### 10.2.2 Security Constraints on Presigned URLs

- **Bind Content-Type to signature:** Go middleware must explicitly mandate the exact MIME type from the LWC (e.g., `image/jpeg`). If an attacker intercepts the URL and uploads `application/x-msdownload`, R2 rejects with `403 SignatureDoesNotMatch`.
- **Tight expiration:** 15–60 minutes maximum.
- **UUID object keys:** Prevent key enumeration or overwrite attacks.

#### 10.2.3 CORS & CSP Configuration for Direct R2 Upload

**R2 bucket CORS rules (required):** The LWC runs within `*.lightning.force.com`, so the browser sends a Preflight OPTIONS request before the cross-origin PUT.
- Allowed origins: the specific Salesforce instance domain
- Allowed methods: `PUT`
- Exposed headers: `ETag` (for upload integrity verification)
- Allowed headers: any custom headers (e.g., `x-amz-server-side-encryption` if KMS is used)

**Salesforce CSP Trusted Sites (required):** The R2 endpoint domain (`https://<ACCOUNT_ID>.r2.cloudflarestorage.com`) must be registered in Setup → CSP Trusted Sites with `connect-src` directive. Failure to configure CSP results in silent browser console blockages — no Apex or network logs.

#### 10.2.4 Binary Streaming from LWC

The JavaScript `File` object from `<input type="file">` implements the `Blob` interface. Pass it directly to the `body` parameter of `fetch()`:

```javascript
// This streams the file chunk-by-chunk — near-zero browser memory footprint
const response = await fetch(presignedUrl, {
    method: 'PUT',
    headers: { 'Content-Type': file.type },
    body: file  // Blob interface — browser streams via OS TCP stack
});
```

This prevents UI locking or out-of-memory crashes. The browser's network stack handles the actual data transfer.

| Component | Responsibility | Memory Profile |
|---|---|---|
| **LWC File Input** | Captures file reference from OS | Negligible (metadata only) |
| **Go Middleware** | Generates SigV4 presigned URL | Minimal (metadata + fast crypto hash) |
| **Browser Network Stack** | Streams binary Blob via fetch (`mode: 'cors'`) | Handled by OS TCP stack |
| **Cloudflare R2** | Ingests binary stream via HTTP PUT | Scalable edge ingestion |

#### 10.2.5 Post-Upload Webhook Flow

1. LWC `fetch()` completes with `200 OK` → file is in R2.
2. LWC sends confirmation payload to Go middleware: `{ "r2_object_key": "<UUID>", "telegram_recipient_id": <int> }`.
3. Go middleware downloads the file from R2 into its own memory, streams it to Telegram via `messages.sendMedia` (MTProto) or `sendDocument`/`sendPhoto` (Bot API).

### 10.3 Why Not Pre-Signed URLs for Inbound/Display?

Exposes bucket names/account IDs to browser. Bypasses CDN caching (each URL unique). Every request hits origin — higher latency and cost. Use HMAC-signed Worker URLs for display instead.

### 10.4 Media Deletion

On Telegram `DeleteMessage` event:
1. R2 `DeleteObject` (S3 API)
2. Cloudflare Cache Purge API (`POST /zones/:zone_id/purge_cache`)
3. Update/delete Salesforce record

### 10.5 Storage Lifecycle

R2 Object Lifecycle Management (GA): up to 1,000 rules per bucket, auto-delete by age, transition to Infrequent Access.

**Policy:** Indefinite for active Salesforce records. 30–60 day TTL for temp/untracked media (prefix `temp/`).

---

## 11. Multi-Messenger Abstraction Layer

### 11.1 Design Principle

The MVP targets Telegram, but the Go middleware must be architected so adding WhatsApp, Facebook Messenger, or any other platform requires only implementing a new adapter — no core restructuring.

### 11.2 Go Interface Design

```go
// messenger.go — Core abstraction

type Platform string

const (
    PlatformTelegram  Platform = "telegram"
    PlatformWhatsApp  Platform = "whatsapp"   // future
    PlatformMessenger Platform = "messenger"   // future
)

// Message is the universal message struct that all adapters produce/consume
type Message struct {
    ID            string
    Platform      Platform
    ChatID        string
    SenderID      string
    SenderName    string
    Text          string
    MediaType     string       // "photo", "video", "voice", "document", ""
    MediaFileID   string       // Platform-specific file reference
    ReplyToID     string
    Timestamp     time.Time
    Direction     Direction    // Inbound | Outbound
    RawPayload    json.RawMessage  // Original platform-specific JSON
}

type Direction string
const (
    Inbound  Direction = "inbound"
    Outbound Direction = "outbound"
)

// MessengerAdapter is what each platform must implement
type MessengerAdapter interface {
    // Platform returns the platform identifier
    Platform() Platform
    
    // Connect establishes the connection (MTProto session, webhook registration, etc.)
    Connect(ctx context.Context, config ConnectionConfig) error
    
    // Disconnect gracefully shuts down
    Disconnect(ctx context.Context) error
    
    // InboundMessages returns a channel of incoming messages
    InboundMessages() <-chan Message
    
    // SendMessage sends a message through this platform
    SendMessage(ctx context.Context, msg OutboundMessage) error
    
    // DownloadMedia downloads a file from the platform and returns a reader
    DownloadMedia(ctx context.Context, fileID string) (io.ReadCloser, MediaMeta, error)
    
    // Status returns the connection health
    Status() ConnectionStatus
}

// ConnectionConfig holds platform-specific config
type ConnectionConfig struct {
    Platform     Platform
    Credentials  map[string]string  // "phone", "api_id", "api_hash", "bot_token", etc.
    ProxyURL     string             // For MTProto IP isolation
    SalesforceID string             // Salesforce Connection record ID
}
```

### 11.3 Telegram Adapter Implementation (MVP)

```go
// telegram_adapter.go — implements MessengerAdapter for BOTH MTProto and Bot API

type TelegramAdapter struct {
    mtprotoClient *telegram.Client     // gotd/td
    botAPI        *botapi.BotClient    // HTTP Bot API client
    mode          string               // "mtproto" | "bot_api"
    messages      chan Message
    // ...
}

func (t *TelegramAdapter) Platform() Platform { return PlatformTelegram }

// Connect initializes either MTProto or Bot API depending on config
func (t *TelegramAdapter) Connect(ctx context.Context, cfg ConnectionConfig) error {
    switch {
    case cfg.Credentials["bot_token"] != "":
        return t.connectBotAPI(ctx, cfg)
    case cfg.Credentials["phone"] != "":
        return t.connectMTProto(ctx, cfg)
    default:
        return errors.New("no valid credentials provided")
    }
}
```

### 11.4 Future Adapter (Example Skeleton)

```go
// whatsapp_adapter.go — future
type WhatsAppAdapter struct {
    // WhatsApp Business API client
    messages chan Message
}

func (w *WhatsAppAdapter) Platform() Platform { return PlatformWhatsApp }
// ... implement all MessengerAdapter methods
```

### 11.5 Database Schema Supports Multi-Platform

The PostgreSQL schema and Salesforce custom objects use generic `messenger_*` naming (not `telegram_*`), and include a `platform` / `Messenger_Platform__c` field. This means no schema changes when adding new platforms.

---

## 12. Infrastructure & Hosting

### 12.1 Geographic Placement

Telegram DCs (DC2, DC4) operate from **Amsterdam, Netherlands** (AS62041). Host Go middleware near Amsterdam.

**Recommended providers:**
- **Hetzner** — Falkenstein/Nuremberg. Ampere ARM instances. Strong EU peering. Cost-effective.
- **DigitalOcean** — Amsterdam (AMS3). Clean IP reputation.
- **AWS Graviton** — `eu-west-1` (Ireland) or `eu-central-1` (Frankfurt). ARM-based.

### 12.2 IP Reputation

Clean, dedicated IPs required. Budget VPS providers with spam-tainted subnets increase ban risk. Deploy residential/ISP proxy pools for per-session MTProto IP isolation.

### 12.3 ARM & CI/CD

Go cross-compilation: `GOOS=linux GOARCH=arm64` (production) + `GOOS=linux GOARCH=amd64` (fallback).

GitHub Actions: native ARM runners or Docker Buildx (avoid QEMU). Produce single multi-arch OCI container manifest.

---

## 13. Database Architecture (Remote PostgreSQL)

### 13.1 PostgreSQL Is the Primary Data Store on the Go Side

PostgreSQL runs as a **dedicated, hosted service** (not embedded/local). It sits next to the Go middleware, either on the same VPS or as a managed service (e.g., Hetzner Managed PostgreSQL, AWS RDS, DigitalOcean Managed DB).

### 13.2 What PostgreSQL Stores (Go Side)

| Data Category | Tables | Notes |
|---|---|---|
| **MTProto Sessions** | `mtproto_sessions` | Active cryptographic keys, auth state, salts. Use **`UNLOGGED` tables** for high-write performance (bypasses WAL). |
| **Bot API Config** | `bot_connections` | Bot tokens (encrypted), webhook URLs, last update ID. |
| **Message Queue** | `outbound_queue`, `inbound_queue` | Durable queues for messages awaiting Salesforce delivery or Telegram sending. |
| **Media References** | `media_files` | R2 object keys, Telegram file IDs, MIME types, sizes. Maps to Salesforce record IDs. |
| **Rate Limit State** | `rate_limit_buckets` | Per-account, per-bot token bucket state (if not using Redis). |
| **Connection Registry** | `connections` | All active messenger connections, mapped to Salesforce org + user IDs. Platform-agnostic. |
| **Proxy Assignments** | `proxy_pool` | Residential proxy IP assignments per MTProto session. |

### 13.3 Key Schema Design

```sql
-- MTProto session storage (UNLOGGED for performance)
CREATE UNLOGGED TABLE mtproto_sessions (
    id              BIGSERIAL PRIMARY KEY,
    connection_id   BIGINT REFERENCES connections(id),
    session_data    BYTEA NOT NULL,        -- Raw gotd session bytes
    dc_id           INT NOT NULL,
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Universal connection registry (multi-platform ready)
CREATE TABLE connections (
    id              BIGSERIAL PRIMARY KEY,
    platform        TEXT NOT NULL,          -- 'telegram', 'whatsapp', etc.
    connection_type TEXT NOT NULL,          -- 'mtproto_user', 'bot_api', 'whatsapp_business'
    sf_org_id       TEXT NOT NULL,          -- Salesforce Org ID
    sf_user_id      TEXT NOT NULL,          -- Salesforce User ID
    sf_connection_id TEXT,                  -- Salesforce record ID (Messenger_Connection__c)
    credentials     JSONB NOT NULL,         -- Encrypted: phone, api_id, bot_token, etc.
    proxy_url       TEXT,
    status          TEXT DEFAULT 'active',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Inbound message queue (Go → Salesforce)
CREATE TABLE inbound_queue (
    id              BIGSERIAL PRIMARY KEY,
    connection_id   BIGINT REFERENCES connections(id),
    platform        TEXT NOT NULL,
    external_msg_id TEXT NOT NULL,
    chat_id         TEXT NOT NULL,
    payload         JSONB NOT NULL,
    status          TEXT DEFAULT 'pending',  -- pending | processing | delivered | failed
    retry_count     INT DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    next_retry_at   TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ
);

-- Outbound message queue (Salesforce → Telegram/etc.)
CREATE TABLE outbound_queue (
    id              BIGSERIAL PRIMARY KEY,
    connection_id   BIGINT REFERENCES connections(id),
    platform        TEXT NOT NULL,
    chat_id         TEXT NOT NULL,
    message_text    TEXT,
    media_file_id   TEXT,
    sf_message_id   TEXT,                    -- Salesforce Messenger_Message__c record ID (for failure sync)
    status          TEXT DEFAULT 'pending',   -- pending | processing | delivered | failed
    error_code      TEXT,                    -- Terminal error code from Telegram (e.g., PEER_ID_INVALID)
    error_detail    TEXT,                    -- Human-readable failure description
    retry_count     INT DEFAULT 0,
    max_retries     INT DEFAULT 5,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    next_retry_at   TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ
);

-- Media file tracking
CREATE TABLE media_files (
    id              BIGSERIAL PRIMARY KEY,
    connection_id   BIGINT REFERENCES connections(id),
    platform        TEXT NOT NULL,
    external_file_id TEXT NOT NULL,          -- Telegram file_id, WhatsApp media ID, etc.
    r2_object_key   TEXT,                    -- Cloudflare R2 key after upload
    mime_type       TEXT,
    file_size       BIGINT,
    sf_record_id    TEXT,                    -- Salesforce Messenger_Message__c ID
    uploaded_at     TIMESTAMPTZ,
    deleted_at      TIMESTAMPTZ              -- Soft delete tracking
);

-- Index for queue workers
CREATE INDEX idx_inbound_queue_pending ON inbound_queue(status, next_retry_at) 
    WHERE status IN ('pending', 'failed');
CREATE INDEX idx_outbound_queue_pending ON outbound_queue(status, next_retry_at) 
    WHERE status IN ('pending', 'failed');
```

### 13.4 Why UNLOGGED Tables for Sessions

PostgreSQL `UNLOGGED` tables skip the Write-Ahead Log (WAL), eliminating disk I/O overhead for ephemeral data. MTProto session state is updated dozens of times per minute (salt rotations, sequence number increments). If the server crashes, session data is lost — but that's acceptable because the user simply re-authenticates. The performance gain is significant.

### 13.5 Queue Processing with SKIP LOCKED

```sql
-- Worker picks up pending messages without blocking other workers
SELECT id, payload FROM inbound_queue
WHERE status = 'pending' AND (next_retry_at IS NULL OR next_retry_at <= NOW())
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

`SKIP LOCKED` allows multiple Go worker goroutines to process the queue concurrently without deadlocks — each worker gets different rows.

### 13.6 When to Add Redis

Start with PostgreSQL only. Add Redis when:
- Horizontal scaling requires distributed rate limiting across multiple Go instances.
- Real-time pub/sub between Go instances is needed (Redis Pub/Sub).
- Session read latency becomes a bottleneck (sub-millisecond reads needed at scale).

---

## 14. Fallback & Queue Design

### 14.1 Salesforce API Throttling (HTTP 429)

When Salesforce returns HTTP 429:
1. **Exponential backoff with jitter.** Don't hammer the API.
2. **Persist undelivered messages** in PostgreSQL `inbound_queue`.
3. **Guarantee ordering** via `external_msg_id` (Telegram message IDs are monotonically increasing per chat).
4. **Replay in order** when limits reset.

### 14.2 Telegram-Side Failures

If Telegram returns `FLOOD_WAIT` for outgoing messages:
1. `floodwait` middleware handles MTProto automatically.
2. For Bot API: update `outbound_queue.next_retry_at` = now + wait_seconds.
3. Worker goroutine respects `next_retry_at` before retrying.

### 14.3 Outbound Message Failure Synchronization

When a Salesforce user sends a message via the LWC, it is written to the `outbound_queue` table and processed by the Go worker pool. Delivery can fail permanently: recipient blocked the bot, account deleted, channel restricts external messages, or the MTProto session requires key renegotiation.

If a message exhausts its maximum retry count, this terminal failure **must be reported back to Salesforce in real-time.** Otherwise, agents operate under a false assumption of successful delivery.

#### 14.3.1 Integration Mechanism: Platform Events (Recommended)

| Mechanism | Strengths | Limitations | Best For |
|---|---|---|---|
| **REST API (PATCH)** | Synchronous commit confirmation | Consumes daily API quotas; higher JSON overhead | Low-volume point-to-point updates |
| **Platform Events** | Scalable pub/sub; decoupled from DB locking; natively consumed by LWC | 1 MB payload cap; hourly delivery limits | High-volume real-time UI updates |
| **Change Data Capture** | Native streaming of SF data changes | Designed for SF→external direction, not inbound | Replicating SF data to external warehouses |

**Decision: Platform Events** — the LWC needs instant visual feedback (e.g., red exclamation mark on the chat bubble) without polling or page refresh.

#### 14.3.2 Failure Workflow Execution

1. **Terminal state identification:** Go worker evaluates the Telegram API response and determines permanent failure (e.g., `USER_BANNED_IN_CHANNEL`, `PEER_ID_INVALID` RPC error).
2. **Event payload construction:** Go middleware constructs JSON conforming to `Message_Delivery_Status__e` schema:
   - Salesforce external ID of the message
   - Final status: `FAILED`
   - Error string from Telegram (for audit trail)
3. **Event publication:** HTTP POST to `/services/data/vXX.X/sobjects/Message_Delivery_Status__e`. Salesforce responds with `201 Created` + `OPERATION_ENQUEUED`.
4. **Real-time UI reflection:** LWC subscribes via `lightning/empApi` to `/event/Message_Delivery_Status__e`. On event receipt, dynamically updates reactive properties → failure indicator renders instantly (no page refresh, no SOQL query).
5. **Database persistence fallback:** An Apex Trigger subscribed to the same Platform Event executes a bulkified DML update on `Messenger_Message__c.Delivery_Status__c` field. This dual-consumption model ensures both instant UI feedback AND long-term data integrity.

#### 14.3.3 Platform Event Schema: Message_Delivery_Status__e

```
tgint__Message_Delivery_Status__e
├── Message_External_ID__c   -- External ID of the Messenger_Message__c record
├── Status__c                -- 'DELIVERED' | 'FAILED' | 'RETRYING'
├── Error_Code__c            -- Telegram RPC error code string
├── Error_Detail__c          -- Human-readable failure reason
└── Timestamp__c             -- ISO 8601 timestamp of the status change
```

---

## 15. Security Checklist

- [ ] All Go server endpoints enforce TLS 1.2+ (SSL Labs scan)
- [ ] DAST scan (OWASP ZAP / Burp Suite) — zero high-severity issues
- [ ] False positives documented in supplementary report
- [ ] HMAC-SHA256 on all Go → Salesforce webhooks
- [ ] Shared secrets in Salesforce Protected Custom Metadata Types
- [ ] No CRM credentials hardcoded in Go source
- [ ] R2 bucket strictly private (no public access)
- [ ] R2 CORS policy configured for Salesforce instance domain (PUT, ETag exposure)
- [ ] CSP Trusted Sites configured for R2 endpoint domain (connect-src directive)
- [ ] Presigned URL generation enforces Content-Type binding to SigV4 signature
- [ ] Presigned URL expiration set to 15–60 minute maximum
- [ ] Media URLs use HMAC-signed tokens with expiration at Cloudflare Worker edge
- [ ] MTProto session keys stored in PostgreSQL, never transmitted to Salesforce
- [ ] Bot tokens encrypted at rest in PostgreSQL
- [ ] Salesforce CSP Trusted Sites configured (Centrifugo, CDN domains)
- [ ] CORS allowlist on both Salesforce and Cloudflare sides
- [ ] Centrifugo JWT tokens short-lived (15 min TTL), with LWC refresh callback
- [ ] Centrifugo shared secret stored in Protected Custom Metadata Type (symmetric) or Self-Signed Certificate HSM (asymmetric)
- [ ] Apex JWT generation uses Base64Url encoding (not standard Base64)
- [ ] Platform Event batching implemented (2-3 sec sliding window, 950 KB threshold)
- [ ] sync.Pool used for batch serialization buffers (prevents GC thrashing)
- [ ] Message_Delivery_Status__e Platform Event published on terminal outbound failures
- [ ] Residential proxy pool for MTProto IP isolation
- [ ] Exponential backoff for all external API retries
- [ ] No automated `my.telegram.org` API ID registration
- [ ] OAuth 2.0 JWT Bearer flow for Go → Salesforce authentication
- [ ] All PostgreSQL credentials injected via environment variables
- [ ] SRP v6a 2FA flow fully implemented for MTProto Cloud Passwords

---

## 16. Key Architectural Decisions Log

| # | Decision | Rationale |
|---|---|---|
| **ADR-1** | Support BOTH MTProto and Bot API bidirectionally | Different use cases require different protocols. Parsing needs MTProto; user interaction needs Bot API. Both send and receive. |
| **ADR-2** | Remote PostgreSQL (not local/embedded) as Go-side data store | Proper hosted database for durability, backups, connection pooling. Not bbolt/SQLite — we need concurrent writes, queue processing, and multi-table relationships. |
| **ADR-3** | Cloudflare R2 + Workers for media | Salesforce storage is expensive. R2 has zero egress. Worker-based HMAC auth is more secure than pre-signed URLs. |
| **ADR-4** | Centrifugo for real-time LWC chat | Salesforce can't host WebSockets. Platform Event limits too restrictive for active chat. |
| **ADR-5** | Single corporate API ID + residential proxy pool | Per-user API IDs violate Telegram ToS. Proxy isolation is the proven strategy. |
| **ADR-6** | HMAC-SHA256 for webhook auth | Simpler than OAuth for high-velocity stateless pushes. Constant-time comparison. |
| **ADR-7** | Hybrid ingestion: Platform Events + Bulk API 2.0 + REST | No single API handles all cases. Events for live UI, Bulk for archives, REST for ad-hoc. |
| **ADR-8** | Pub/Sub API (gRPC) for Go ↔ Salesforce event streaming | Replaces legacy CometD. ~7-10x faster, bidirectional, pull-based flow control. Excellent Go gRPC support. |
| **ADR-9** | Hetzner (Europe) as primary hosting | Proximity to Telegram DCs in Amsterdam. Clean IPs. ARM available. Cost-effective. |
| **ADR-10** | Messenger-agnostic Go interface design | MVP is Telegram, but WhatsApp/Messenger/Viber coming. Adapter pattern means new platforms = new Go files, not restructuring. |
| **ADR-11** | 2GP Managed Package (second-generation) | Salesforce recommends 2GP for all new packages. Better CI/CD, scratch org support, versioning. |
| **ADR-12** | Generic `Messenger_*` naming for all objects/tables | Not `Telegram_*`. Supports multi-platform from day one without schema migrations when adding platforms. |
| **ADR-13** | Centrifugo JWT auth via Apex-generated tokens (symmetric HMAC-SHA256 default, asymmetric RSA-SHA256 for cross-org) | Centrifugo is stateless — needs cryptographic trust from Salesforce session context. Short-lived JWTs with LWC refresh callback prevent stale connections. |
| **ADR-14** | Direct-to-R2 outbound media upload via SigV4 presigned URLs | Avoids routing binary files through Salesforce storage tier (2 GB quota, DB bloat). LWC streams to R2 with near-zero memory footprint. |
| **ADR-15** | SRP v6a implementation for MTProto 2FA via `gotd/td` crypto/srp package | PBKDF2 with 100K iterations must run on Go backend (browser JS would lock UI). Zero-knowledge proof — password never transmitted. |
| **ADR-16** | Platform Events for outbound delivery failure sync (`Message_Delivery_Status__e`) | Real-time UI feedback for agents + durable persistence via dual-consumer pattern (LWC empApi + Apex Trigger). |
| **ADR-17** | sync.Pool + streaming JSON encoder for batch byte tracking | Prevents GC thrashing under high-throughput bursts. Individual struct-level byte tracking avoids full-slice re-serialization. |

---

## 17. Glossary

| Term | Definition |
|---|---|
| **MTProto** | Telegram's core cryptographic protocol. Full event fidelity, 2GB file limits. Requires phone number auth. |
| **Bot API** | Telegram's HTTP abstraction for bots. Simpler but limited (20MB downloads, filtered events). Token-based. |
| **gotd/td** | Pure Go MTProto 2.0 client library. ~150KB per idle client. Our core Telegram engine. |
| **GoTGProto** | Higher-level wrapper around `gotd/td` for session management and handler routing. |
| **Salesforce Org** | One customer's Salesforce instance. Has its own data, code, and config. Like a tenant. |
| **Apex** | Salesforce's Java-like server-side language. Runs on Salesforce servers, not yours. |
| **SOQL** | Salesforce Object Query Language. Like SQL but simpler, no JOINs. |
| **LWC** | Lightning Web Components. JavaScript UI framework (like React). Runs in the user's browser inside Salesforce. |
| **Custom Object** | A database table you define in Salesforce. Suffixed with `__c`. |
| **Managed Package** | A distributable bundle of Salesforce components (objects, code, UI). Installed via AppExchange. |
| **2GP** | Second-Generation Managed Package. Salesforce's recommended packaging format. |
| **Namespace** | A unique prefix for your managed package (e.g., `tgint__`). Prevents naming conflicts. |
| **Governor Limits** | Hard runtime quotas enforced by Salesforce (100 SOQL queries/transaction, 150 DML, etc.). Exceeding = unhandleable exception. |
| **Platform Events** | Salesforce's pub/sub event bus. 1MB payload, hourly delivery allocations. |
| **Pub/Sub API** | Salesforce's gRPC-based API for event streaming. Replaces CometD for external systems. |
| **CometD / empApi** | Legacy event subscription protocol. Still used in LWC browser-side via `lightning/empApi`. |
| **Connected App** | OAuth2 client registration in Salesforce. Defines how your Go server authenticates. |
| **Protected Custom Metadata Type** | Encrypted key-value config in Salesforce, accessible only within your managed package. For secrets. |
| **Composite App** | AppExchange classification for apps with external server components. Requires extra security review. |
| **Centrifugo** | Open-source Go WebSocket server. Bridges real-time messages to LWC, bypassing Salesforce limits. |
| **Cloudflare R2** | S3-compatible object storage. Zero egress fees. For media files. |
| **Cloudflare Workers** | Serverless edge compute. Our reverse proxy for secure, cached media delivery. |
| **HMAC** | Hash-based Message Authentication Code. Used for webhook verification (SHA-256) and media URL signing. |
| **FLOOD_WAIT** | MTProto rate limit error. Contains seconds to wait before retrying. |
| **errgroup** | Go sync primitive (`golang.org/x/sync/errgroup`). Bounded goroutine pools with error propagation. |
| **UNLOGGED tables** | PostgreSQL tables that skip WAL. Fast writes for ephemeral data (sessions). Not crash-safe. |
| **SKIP LOCKED** | PostgreSQL row-locking that skips locked rows. Enables concurrent queue processing without deadlocks. |
| **CSP Trusted Sites** | Salesforce config allowing LWC JavaScript to connect to external domains. |
| **CORS Allowlist** | Salesforce config permitting cross-origin requests from external domains. |
| **DAST** | Dynamic Application Security Testing. Required for AppExchange security review. |
| **PADA** | Partner Application Distribution Agreement. Business agreement for AppExchange publishing. |
| **JWT Bearer Flow** | OAuth 2.0 server-to-server auth flow using signed JSON Web Tokens. Recommended for Go → Salesforce. |
| **SRP v6a** | Secure Remote Password protocol version 6a. Zero-knowledge proof mechanism used by Telegram for 2FA (Cloud Password) verification. Password is never transmitted — only cryptographic proof of knowledge. |
| **PBKDF2** | Password-Based Key Derivation Function 2. Used in MTProto SRP with HMAC-SHA512 over 100,000 iterations for brute-force resistance. |
| **Presigned URL** | A time-limited, cryptographically signed URL that grants temporary authorization to perform a specific operation (e.g., PUT upload) on an S3-compatible bucket without embedding IAM credentials in the browser. |
| **SigV4** | AWS Signature Version 4. The signing protocol used by Cloudflare R2 (S3-compatible) to authenticate presigned URL requests. |
| **Base64Url** | URL-safe variant of Base64 encoding (RFC 4648). Replaces `+` with `-`, `/` with `_`, strips `=` padding. Required for JWTs. Salesforce Apex does NOT produce this natively — manual character substitution required. |
| **sync.Pool** | Go standard library construct for reusing allocated objects across goroutines, reducing GC pressure during high-throughput operations like batch JSON serialization. |

---

*Document compiled: March 2026. Updated with edge case specifications for Centrifugo JWT auth, direct-to-R2 outbound media uploads, MTProto SRP v6a 2FA, outbound failure synchronization, and memory-optimized dynamic batching. Sources: Salesforce developer documentation, Telegram API documentation, gotd/td GitHub, Cloudflare R2 docs, Salesforce Pub/Sub API docs, Centrifugo authentication docs, and architectural research.*
