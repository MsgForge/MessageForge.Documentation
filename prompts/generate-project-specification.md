You are a senior technical writer producing the canonical project specification for MessageForge — a multi-messenger to Salesforce CRM integration platform.

Your task: write a single, exhaustive Product & Technical Specification document. This document must be detailed enough that any AI agent (with zero prior context) can implement any part of the system correctly without asking clarifying questions.

## Source material to read first

Before writing, read ALL of the following files in full:

- `MessageForge.Documentation/architecture/architecture.md`
- `MessageForge.Documentation/architecture/adr.md`
- `MessageForge.Documentation/reference/sf-data-model.md`
- `MessageForge.Documentation/reference/postgres-schema.md`
- `MessageForge.Documentation/reference/salesforce-guide.md`
- `MessageForge.Documentation/reference/governor-limits.md`
- `MessageForge.Documentation/reference/security-checklist.md`
- `MessageForge.Documentation/plans/mvp-implementation-plan.md`
- `MessageForge.Documentation/plans/future-multi-messenger.md`
- `MessageForge.Documentation/reviews/2026-03-29-architecture-review.md`
- `MessageForge.Documentation/research/complete-technical-specification.md`
- `CLAUDE.md` (root)

---

## Document structure

Write the document with these exact sections:

---

### 1. Project Overview

- What MessageForge is (one paragraph — product-level)
- The core problem it solves (CRM agents managing messaging channels without leaving Salesforce)
- MVP scope: Telegram (Bot API + MTProto). Future: WhatsApp, Viber, Twilio, Apple Messages
- Target users: Salesforce org admins (configure channels) + agents/reps (use chat)
- Distribution: Salesforce AppExchange 2GP Managed Package (namespace: `tgint__`)

---

### 2. High-Level Architecture

- System diagram (ASCII) showing: Messaging Platforms → Go Middleware → PostgreSQL, Go Middleware → Salesforce (REST + Platform Events), LWC ↔ Salesforce
- Component responsibility table: Go Middleware, PostgreSQL, Salesforce Apex, Salesforce LWC
- Authentication map: Go→SF (OAuth 2.0 JWT Bearer), SF→Go (HMAC-SHA256 `X-Signature`), Browser→SF (session-based)
- Key architectural decisions (summarize all 22 ADRs as a table: ADR number, decision, why it matters for implementation)

---

### 3. Go Middleware — Full Specification

#### 3.1 Repository structure
Describe exact directory layout under `MessageForge.Backend/` with purpose of each package.

#### 3.2 Configuration
All environment variables with types, required/optional, defaults, validation rules.

#### 3.3 Telegram: dual-protocol design
- MTProto (via gotd/td + GoTGProto): use cases, auth flow (5-phase state machine: PhoneInput → OTPSent → OTPInput → TwoFA → Success), SRP v6a 2FA, session storage in PostgreSQL UNLOGGED tables
- Bot API webhook: use cases, webhook registration, update parsing
- Unified `adapter.Adapter` interface both must implement
- Rate limiting middleware chain: per-account bucket → floodwait handler → RPC

#### 3.4 Salesforce integration (Go side)
- OAuth 2.0 JWT Bearer token generation and refresh
- Platform Event publishing: `Inbound_Message__e`, `Session_Status__e`, `Message_Delivery_Status__e`
- REST API: inbound bulk endpoint, ContentVersion multipart upload, ContentVersion streaming download
- Data ingestion pipeline: batching strategy (950KB threshold, 2-3s sliding window), sync.Pool for buffers

#### 3.5 Media pipeline
- Inbound: Telegram → Go downloads (MTProto for >20MB, Bot API for ≤20MB) → ContentVersion REST upload → create `Messenger_Attachment__c`
- Outbound: LWC file upload → ContentVersion → Go streams download from SF → sends to Telegram

#### 3.6 PostgreSQL schema
List all tables with columns, types, indexes, and the UNLOGGED + backup snapshot strategy for MTProto sessions.

---

### 4. Salesforce Package — Full Specification

#### 4.1 Data model
Complete list of all 21 custom objects + 2 Protected CMTs + 3 Platform Events. For each object: purpose, all fields with type and description, key relationships.

#### 4.2 Apex classes and triggers
List all Apex classes and triggers with their responsibility, key methods, bulkification rules, governor limit considerations.

#### 4.3 Permission model
Three permission sets: `Messenger_Admin`, `Messenger_Agent`, `Messenger_Viewer`. Table showing which objects each can CRUD.

#### 4.4 Security
- HMAC-SHA256 validation in `HMACValidator.cls` (constant-time comparison)
- AES-256 encryption for credentials via `EncryptionService.cls` and Protected CMT `Encryption_Key__mdt`
- Apex Managed Sharing via `ChannelAccessService.cls` (respects `Channel_User_Access__c`)
- FLS-resilient field access: use `stripInaccessible` + server-side fallback

---

### 5. Salesforce Frontend — LWC Component Specifications

This is the most critical section. Write it at the level of a detailed UI/UX specification + implementation guide. An agent must be able to build each component from this spec alone.

---

#### 5.1 `channelSetupWizard` — Channel Connection Manager

**Purpose:** Admin UI for connecting, managing, and archiving messenger channels. Replaces any need to touch raw Salesforce records directly.

**Access:** Requires `Messenger_Admin` permission set. Available as a Lightning App Page tab and embeddable in any App.

**Layout:**
- Full-page component with a top toolbar and a main content area
- Top toolbar: page title ("Messenger Channels"), search bar (filters channel list by name, type, status), "Connect New Channel" primary button (right-aligned)
- Main content area: filterable, scrollable list of channel cards

**Channel list:**
- Each card shows: platform icon (Telegram bot / Telegram user / WhatsApp / Viber / etc.), display name, channel type label, current status badge (draft / active / suspended / archived — color coded: gray / green / yellow / red), session status indicator (online dot, offline dot, connecting spinner, auth_required warning icon), last connected timestamp
- Cards are grouped by status: Active first, then Draft, then Suspended, then Archived
- Within each group, sorted by last connected date descending
- Filtering: multi-select filter bar below toolbar. Filter by: platform (Telegram, WhatsApp, Viber, Twilio, Apple, Vonage, Bird), status (draft, active, suspended, archived), assigned user/group (typeahead lookup)
- Filters persist in component state (not URL). Chip-style filter tags appear below toolbar showing active filters with X to remove each
- Infinite scroll or pagination (page size 20). Load more button at bottom
- Empty state: illustration + "No channels connected yet. Click 'Connect New Channel' to get started."

**Connect New Channel — wizard flow:**

Step 1 — Select Platform:
- Grid of platform cards (icon + name): Telegram Bot, Telegram User (MTProto), WhatsApp Cloud API, Viber, Twilio SMS, Vonage SMS, Bird, Apple Messages for Business
- Each card has a short description (one sentence) of the use case
- Selecting a card highlights it and activates Next button

Step 2 — Platform App (account credentials):
- Shown only for platforms that have App objects (Telegram, WhatsApp, Twilio, Vonage, Bird)
- Dropdown: "Select existing App" OR "Create new App"
- If creating new: inline form with the App's credential fields (e.g., for Telegram App: API ID, API Hash — with help text linking to my.telegram.org)
- Credentials are masked on entry (password type inputs)
- "Test Connection" button validates credentials before proceeding

Step 3 — Channel Credentials:
- Form fields specific to the selected platform and channel type
- Telegram Bot: Bot Token field + "Validate Token" button (calls Go middleware to verify token via Bot API)
- Telegram User: Phone number field → triggers Go middleware auth flow → OTP input appears inline → if 2FA enabled, password field appears
- WhatsApp: Phone Number ID, WABA ID, Access Token
- Other platforms: their respective fields
- All credential inputs masked. "Show" toggle on each
- Inline validation with helpful error messages (not just "invalid", but "This phone number is already registered in another channel")

Step 4 — Channel Settings:
- Display Name (text input, required)
- Assign to Users/Groups: multi-select with typeahead. Each assignee gets an Access Level dropdown (owner / read_write / read_only). Min 1 assignee required
- Primary Assignee toggle (radio among assignees) — for lead routing
- Active toggle (default on)

Step 5 — Review & Save:
- Summary of all entered values (credentials masked)
- "Create Channel" button
- On success: channel card appears in list with status "draft" → transitions to "connecting" → "active" (real-time via Platform Event subscription `Session_Status__e`)
- On error: inline error message with retry option

**Channel card actions (contextual menu):**
- Edit Settings (opens Step 4 in an edit modal)
- Reconnect (triggers Go middleware to re-establish session)
- Rotate Credentials (opens Step 3 in edit mode, marks old credentials as rotated in audit log)
- Suspend (sets Status__c = suspended, stops Go connection)
- Archive (sets Status__c = archived, confirmation dialog required)
- View Audit Log (opens side panel with `Channel_Audit_Log__c` records for this channel)
- Delete (only for draft/archived, requires typed confirmation)

**Real-time updates:**
- Component subscribes to `Session_Status__e` via empApi on connectedCallback
- Session status indicator updates live without refresh
- If a channel goes to `auth_required` or `mfa_required`: a yellow banner appears on that card with a "Re-authenticate" action that opens the auth step inline

**Implementation notes:**
- Apex controller: `ChannelSetupController.cls` — all DML in methods annotated `@AuraEnabled(cacheable=false)`
- Credentials never returned to LWC after save (write-only). Display masked placeholder `••••••••` instead
- Go middleware calls triggered from Apex `Queueable` (to avoid LWC callout limits)
- Use `lightning-card`, `lightning-badge`, `lightning-button`, `lightning-combobox`, `lightning-input`, `lightning-spinner` SLDS components
- Keyboard navigation: Tab through filter chips, channel cards support Enter to open detail

---

#### 5.2 `messengerChat` — Main Agent Chat Interface

**Purpose:** Primary working interface for agents. Mirrors the Telegram Desktop layout. Left panel: list of all chats. Right panel: active chat thread.

**Access:** Requires `Messenger_Agent` or higher. Available in Service Console, Account/Contact record pages, and as standalone App Page.

**Overall layout:**
- Two-column layout. Left column: fixed 340px width, not resizable on mobile (collapses to full-screen list → full-screen chat navigation). Right column: fills remaining width
- Minimum supported viewport: 1024px wide. Mobile: single-panel with back button
- No page header. Full-height component.

---

**Left panel — Chat List:**

Header:
- "Chats" title (h1, 16px bold)
- Search icon button (expands inline search bar when clicked)
- Compose/New Chat icon button (only shown for channels with `Supports_Outbound__c = true`)

Search bar (expanded state):
- Full-width text input, placeholder "Search chats..."
- Filters chat list in real-time (debounced 300ms) by: chat title, last message preview, sender name
- Clear (X) button to collapse search bar

Chat list items:
- Each item: channel platform icon (16px, left), contact/chat avatar (32px circle with initials fallback, left), chat title (bold, truncated), last message preview (gray, truncated to one line), timestamp (right-aligned, relative: "just now", "2m", "14:32", "Mon", "Apr 3"), unread count badge (red pill, right). If unread = 0, badge hidden
- Items sorted by `Last_Message_At__c` descending
- Unread chats: left border accent (3px blue), title bold, background slightly highlighted
- Active/selected chat: highlighted background (SLDS `--lwc-colorBackgroundSelectedHover`)
- Long-press / right-click context menu: Mark as Read, Assign to Me, Resolve, Archive

Filter bar (bottom of left panel, always visible):
- Row of filter controls:
  - "Source" dropdown: All Channels | [list of platforms: Telegram, WhatsApp, Viber...]
  - "Channel" dropdown (depends on Source selection): All | [list of `Messenger_Channel__c` records the user has access to, grouped by platform]
  - "Status" segmented control or dropdown: All | Open | Pending | Resolved | Archived
  - "Assigned" toggle: All / Mine (filter to chats where `Assigned_User__c = currentUser`)
- Filter state stored in component. Cleared by "Reset Filters" link that appears when any filter is active
- Filter bar itself is horizontally scrollable if filters overflow on small screens

Infinite scroll: load 30 chats at a time. "Loading..." spinner at bottom during fetch. Virtualized list (only render visible items).

Empty states:
- No chats at all: "No conversations yet. When a customer messages you, it will appear here."
- No chats matching filters: "No chats match your current filters." + "Reset Filters" link

Real-time updates:
- Subscribe to `Inbound_Message__e` via empApi. On new message: find matching chat in list, update preview text, timestamp, increment unread counter, re-sort list (or animate item to top). If chat doesn't exist in list (new conversation): prepend it

---

**Right panel — Chat thread:**

Empty state (no chat selected):
- Centered illustration + "Select a conversation to start chatting"

Chat header (when chat selected):
- Back button (mobile only, left)
- Contact avatar + chat title (bold) + chat type label (private / group / channel)
- Platform icon + channel name (smaller, below title)
- Status pill: Open / Pending / Resolved (color coded)
- Actions row (right-aligned): Assign button (opens user picker), Status dropdown (Open / Pending / Resolved), More (⋮) menu
  - More menu: View Contact/Lead record (opens SF record in new tab), Archive Chat, View Channel

Message thread:
- Scrollable area, messages ordered oldest → newest, scroll anchored to bottom on new messages
- Auto-scroll to bottom when: user is already at bottom (within 100px) AND new message arrives. If user has scrolled up: do NOT auto-scroll; show "↓ New message" floating button
- Date separators between messages from different calendar days (centered, gray, e.g., "Monday, April 7")

Message bubbles:
- Inbound (from contact): left-aligned, gray background, sender name above bubble (for group chats), platform avatar (16px) left of bubble
- Outbound (sent by agent): right-aligned, blue background (SLDS brand color), no sender label
- Timestamp: below bubble, small gray text, right-aligned within bubble. Show: HH:MM. On hover: show full datetime tooltip
- Delivery status icon (outbound only): clock (pending) → single checkmark (sent) → double checkmark (delivered) → double checkmark blue (read) → red exclamation (failed). Failed: clicking opens error detail popover with error code + retry button
- Edited indicator: "(edited)" text appended to message text, small gray
- Reply-to: if message is a reply, show quoted snippet above bubble (gray border left, truncated to 2 lines, italic)

Message types:
- text: rendered as plain text. URLs auto-linked (use `lightning-formatted-url` or native `<a target="_blank" rel="noopener noreferrer">`). Emoji rendered natively
- photo: inline image, max 300px wide, click to open full-size in `messengerMediaViewer` lightbox. Lazy loaded
- video: thumbnail + play button overlay. Click to open in `messengerMediaViewer`. Duration badge (bottom right of thumbnail)
- voice/audio: inline waveform player (HTML5 `<audio>`) with play/pause, seek bar, duration, playback speed toggle (1x/1.5x/2x)
- document: file icon (based on MIME type) + filename + file size + download button
- sticker: rendered as image, max 180px, no bubble background
- album (multi-media): grid layout. 2 photos: side by side. 3 photos: 2+1 or 1+2. 4+: 2×N grid. Max display 4; if more, overlay "+N more" on last tile. Click any tile to open lightbox on that item with prev/next navigation
- template: rendered as template preview card with header, body, footer, buttons (non-clickable in chat history)
- system/status messages (e.g., "Chat assigned to Jane"): centered, gray italic, no bubble

Compose area (bottom of right panel):
- Visible only when chat status is Open or Pending (hidden for Resolved/Archived)
- Attach button (paperclip icon): opens file picker → uses `lightning-file-upload` → on upload success creates `Messenger_Attachment__c` + ContentVersion. Supported types: image/*, video/*, application/pdf, and common document types. Max file size: 50MB (Bot API) or 2GB (MTProto — show warning if >50MB on bot channels)
- Text input: multi-line, auto-expands up to 5 lines, then scrolls. Placeholder: "Type a message..."
- Emoji picker button (smiley icon): opens emoji grid popover
- Template button (only for channels with templates configured): opens template selector modal
- Send button: right of input. Disabled when input empty AND no attachment. Keyboard shortcut: Enter to send, Shift+Enter for newline
- Typing indicator: displayed above compose area when active (but implementing actual typing detection via Telegram MTProto is Phase 2 — for MVP, this area is reserved)

Assignment panel (side panel, opens from "Assign" button):
- Slide-in panel (right side of chat area)
- Search users/groups typeahead
- Current assignees listed with access level
- Reassign primary assignee radio
- "Save Assignment" button → updates `Channel_User_Access__c` + `Assigned_User__c` on `Messenger_Chat__c`

Apex backend: `MessengerController.cls`
- `getChats(filters)` — SOQL with dynamic WHERE clause. Returns `Messenger_Chat__c` + last message preview + unread count
- `getMessages(chatId, offset, limit)` — paginated, ordered by `Sent_At__c` ASC
- `sendMessage(chatId, text, attachmentId)` — creates `Messenger_Message__c` (direction=outbound, status=pending) → Queueable callout to Go middleware
- `markAsRead(chatId)` — zeroes `Unread_Count__c`
- `updateChatStatus(chatId, status)` — updates `Status__c`
- `assignChat(chatId, userId)` — updates `Assigned_User__c`
All methods enforce FLS via `stripInaccessible` + explicit field-level checks.

---

#### 5.3 `messengerMediaViewer` — Media Lightbox

**Purpose:** Full-screen media viewer for images, video, and documents attached to messages.

**Trigger:** Called by `messengerChat` when user clicks any media item.

**Layout:**
- Full-screen overlay (modal), dark semi-transparent background
- Close button (X) top-right, also closes on Escape key or click outside
- Navigation arrows (prev/next) for album navigation. Hidden if only one item
- Keyboard: Left/Right arrows to navigate, Escape to close
- Center: media display area
  - Images: fit to viewport (max 90vh × 90vw). Pinch-to-zoom on touch. Double-click to zoom 2x, double-click again to reset
  - Video: HTML5 `<video>` player with controls, autoplay muted, loop off
  - Audio: HTML5 `<audio>` player
  - PDF: native browser embed (`<embed type="application/pdf">`) with fallback download link
  - Other documents: filename + file type icon + "Download" button
- Bottom bar: filename, file size, download button, "View in Salesforce Files" link (opens ContentVersion record)
- Media accessed via `/sfc/servlet.shepherd/version/renditionDownload?rendition=THUMB720BY480&versionId={Id}` for thumbnails and full URL for originals. No CSP Trusted Sites needed (native SF domain)

---

### 6. Data Flows — End-to-End Traces

Write step-by-step traces (numbered steps, mentioning exact class/method/table at each step) for:

1. Inbound message: Telegram user sends message → agent sees it in chat
2. Outbound message: agent types and sends → Telegram user receives it
3. Inbound media (photo): Telegram photo → agent sees image in chat
4. Channel connection: admin connects Telegram Bot channel for the first time
5. Real-time delivery status: agent sends message → delivery status icon updates live

---

### 7. Key Constraints and Rules

- Apex governor limits that affect every design decision (list the critical ones from governor-limits.md)
- Platform Event limits (50,000/24h delivery) and when to fall back to Bulk API 2.0
- Telegram rate limits (Bot API: 30 msg/s global, 1 msg/s per DM, 20 msg/min per group; MTProto: per API ID + IP)
- File size limits (Bot API: 20MB download / 50MB upload; MTProto: 2GB both)
- Security rules that must never be violated (hardcoded secret check, FLS enforcement, HMAC validation, etc.)

---

### 8. What is NOT in scope (MVP)

- WhatsApp, Viber, Twilio, Vonage, Bird, Apple Messages (architecture supports them, implementation deferred)
- Pub/Sub API gRPC (Platform Events used instead until 50K/24h limit is reached)
- AI message classification / auto-responses
- Salesforce Einstein integration
- Bulk message campaigns / broadcasts
- Multi-language / localization

---

## Output requirements

- Format: Markdown
- Length: as long as necessary — do not truncate any section. This document is the source of truth
- Tone: technical, precise. Written for a developer/agent, not an end user
- Code examples: include only when they clarify a non-obvious constraint or pattern
- Output the complete document to: `MessageForge.Documentation/project-specification.md`
