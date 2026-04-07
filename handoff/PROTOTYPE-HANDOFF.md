# MessageForge LWC Prototype — Handoff Document

**Artifact:** `index.html` (single-file, no build step)
**Purpose:** Clickable design prototype of a multi-messenger CRM integration that will eventually ship as Salesforce Lightning Web Components. This doc is the input for the next agent that will iterate on or productionize the prototype.

---

## 1. What this prototype is (and isn't)

**It is:**
- A visual and interaction spec for the layouts, flows, and component boundaries.
- A self-contained HTML file with in-memory state, runnable by opening it in a browser.
- A reference for screen structure, state shape, and seed data the developer should match.

**It is not:**
- A styling spec. Most CSS is custom (hand-rolled to look like SLDS) — see §9.
- A real LWC. There is no Salesforce, no Apex, no Platform Events, no MCP.
- Backed by persistence. Reloading the page resets everything.

---

## 2. Tech constraints

- Single `index.html` file. No build step, no npm, no bundler.
- Vanilla JS only. No frameworks.
- SLDS loaded from CDN: `https://unpkg.com/@salesforce-ux/design-system/assets/styles/salesforce-lightning-design-system.min.css`
- All state lives in a single `state` object inside one `<script>` block.
- Mobile-aware but desktop-first (Lightning Console-style layout).
- Mock/seed data only — clearly fake but realistic (Cyprus-based: `+357 99 …`, Limassol references, etc.).

---

## 3. File anatomy

The file is roughly:

| Lines      | Section                                                          |
|------------|------------------------------------------------------------------|
| `<head>`   | SLDS CDN link + ~280 lines of custom CSS                         |
| Shell      | Left nav rail + top tabbar + workspace                           |
| Screen 1   | Channels list (sub-view: list) and Wizard (sub-view: wizard)     |
| Screen 2   | Conversations (two-pane chat)                                    |
| Screen 3   | Lead Record (embedded chat with channel switcher)                |
| Screen 4   | Settings (placeholder)                                           |
| Modals     | Soft-delete modal                                                |
| Utility    | Bottom utility bar + Messages popup (global, fixed)              |
| Toast stack| Top-center toast container                                       |
| `<script>` | ~700 lines of JS — state, render functions, event handlers       |

---

## 4. Global UI

### Nav rail (left)
Vertical strip, dark blue (`#032d60`), 56px wide. Four tab buttons (App Launcher icon is decorative):
- Channels
- Conversations
- Lead Record (demo)
- Settings

### Tabbar (top)
Horizontal tabs mirroring the rail. Both control the same `setTab(tab)` function.

### Workspace
Full-width below the tabbar. Renders one of the four screens based on `state.activeTab`.

### Bottom utility bar
Fixed at the bottom of the viewport on every screen. Modeled after the Salesforce Utility Bar (and the SMS Ninja screenshot from §10).

- `💬 Messages [N]` — N is the unread count across all conversations, recomputed by `refreshUtilityBadge()`.
- `History`, `Notes`, `To Do` — placeholder buttons that fire toasts.
- Right side: prototype meta-label.

Clicking **Messages** opens the popup panel (see §6).

### Toast stack
Top-center, SLDS `slds-notify_toast` markup. Two variants: `success` (green) and `error` (red). Auto-dismiss at ~3s.

---

## 5. Screens

### 5.1 Channels (Screen 1)

Has two sub-views controlled by `state.channelView`: `'list'` or `'wizard'`. Switching is `showWizardView(boolean)`.

#### Sub-view: List
- Search box, channel-type dropdown filter, status dropdown filter, `+ New Channel` button.
- 7-column SLDS-style table: Name (with sub-line), Type (icon + label), Status (pill), Active (checkbox), Messages (24h), Last Activity, Actions.
- Action buttons are status-aware:
  - `online` → **Open** (jumps to Conversations tab)
  - `auth_required` → **Reauth** (toast stub)
  - `error` → **Logs** (toast stub)
  - `draft` → **Resume Setup** (jumps into wizard at step 2)
- Edit button (hidden for `draft`) opens the wizard at step 3 prefilled.
- Delete button uses the soft-delete modal.
- Empty state if `state.channels.length === 0`.
- Recent Audit Events panel below the table is static seed data (4 rows).

#### Sub-view: Wizard
Five-step linear wizard with a numbered step header (active step blue, completed steps green).

| Step | Name                          | Notes                                                     |
|------|-------------------------------|-----------------------------------------------------------|
| 1    | Channel Type                  | 9 cards in a 3-col grid; 6 active, 3 disabled (Phase 2)  |
| 2    | App Credentials               | App Name, API ID, API Hash (masked), Owner Email          |
| 3    | Channel Config                | Channel Name, Handle, Token (masked), Assignee, Auto-link |
| 4    | Test Connection               | Run Test button reveals a static success block            |
| 5    | Activate & Register Webhook   | Read-only webhook URL, HMAC secret, sharing dropdown      |

Footer has Back / Cancel / Next. The Next button on step 5 changes to **✓ Activate** and calls `saveChannel()`, which either creates a new row in `state.channels` or updates the existing one (when `wizard.editingId` is set), fires a toast, and closes the wizard.

### 5.2 Conversations (Screen 2)

Two-pane layout. Left pane is the conversation list, right pane is the active chat thread.

#### Left pane
1. **Channel filter block** (top):
   - Chip row: All / 📨 Telegram / 💚 WhatsApp / 💬 SMS / 💜 Viber. Updates `state.channelTypeFilter`.
   - Dropdown: lists every channel matching the active type, plus "All channels". Updates `state.channelIdFilter`.
   - The dropdown re-renders when the chip selection changes (cascading filter).
2. **Search input** with the SLDS search icon.
3. **All / Unread chips** as the secondary filter.
4. **Scrollable conversation list**, each row:
   - Colored avatar (cycles through 6 colors by index)
   - Name with channel-type icon prefix
   - Relative timestamp
   - Last message preview (handles text / image / file / system)
   - Unread badge

#### Right pane
- **Header:** avatar, name, source badge (`📨 via Sales Bot — EU`), online/last-seen, "Linked to Lead: …" pill.
- **Message area:** SLDS-blue outgoing bubbles right-aligned, white incoming bubbles left-aligned. ✓ / ✓✓ on outgoing. Image placeholder, file attachment card, and centered system messages.
- **Typing indicator** below the messages, animated dots.
- **Composer:** attachment + emoji icon buttons + textarea (Enter sends, Shift-Enter newline) + Send button.

Sending appends a bubble, fakes a typing indicator for ~2s, then marks the message read.

### 5.3 Lead Record (Screen 3)

Mock Salesforce Lead record page demonstrating where the embedded chat LWC sits.

- **Lead header:** large avatar, name (Maria Constantinou), sub-line, action buttons (Edit / Convert / New Task).
- **Lead Details panel:** 8-field grid (Phone, Email, Status, Lead Source, Owner, Industry, Annual Revenue, Created).
- **Embedded chat panel:**
  - Top: channel-switcher tab strip — one tab per channel this Lead has been contacted on (`state.leadChannels = ['c1','c2','c3']`).
  - Each tab swaps the message thread + composer to the channel-specific thread (`state.leadThreads[channelId]`).
  - Composer placeholder reflects the active channel name.
  - Sending appends a bubble to that channel's thread only.
- Legend below flags `[OPEN §10.1]` (cross-channel timeline merging).

### 5.4 Settings (Screen 4)
Placeholder card. Out of prototype scope.

---

## 6. Messages popup (utility bar)

Toggled by `toggleUtilPopup()`. Anchored above the utility bar at `bottom: 32px; left: 12px`.

- **Header:** dark blue, with title and minimize button.
- **Tabs:** New Chats / Live Chats / Pending. Implemented in `renderUtilPopup()`:
  - `new` — conversations where `unread > 0`
  - `live` — conversations whose last message is outbound
  - `pending` — conversations whose last message is inbound
- **Rows:** small avatar, name, channel name, preview, unread badge, **Open chat** + **Open lead** action buttons.
  - Open chat → switches to Conversations tab + selects that thread.
  - Open lead → switches to Lead Record tab (all leads route to Maria for the demo).
- **Quick reply input** at the bottom: sends to the first row of the current view, fires a toast, refreshes the badge.

---

## 7. State model

A single global `state` object. All renders read from it; all event handlers mutate it then call the relevant `render*()`.

```js
state = {
  // Navigation
  activeTab: 'channels',           // 'channels' | 'conversations' | 'lead' | 'settings'
  channelView: 'list',             // 'list' | 'wizard'

  // Channel CRUD
  channels: [
    { id, name, sub, type, status, active, msgs24, lastActivity, errorNote? }
  ],
  wizard:    { step: 1, type: 'tg-bot', editingId: null },
  deletingId: null,

  // Conversations
  activeConvId:      'cv1',
  filter:            'all',        // 'all' | 'unread'
  channelTypeFilter: '',           // '' or a channel type key
  channelIdFilter:   '',           // '' or a specific channel id
  conversations: [
    { id, name, channelId, unread, lastTime, online, messages: [...] }
  ],

  // Lead page
  leadActiveChannel: 'c1',
  leadChannels:      ['c1','c2','c3'],
  leadThreads: {
    c1: [ ...messages ],
    c2: [ ...messages ],
    c3: [ ...messages ]
  },

  // Utility bar popup
  utilOpen: false,
  utilTab:  'new'                  // 'new' | 'live' | 'pending'
}
```

### Message shape
```js
{
  id,
  dir: 'in' | 'out' | 'sys',
  type: 'text' | 'image' | 'file',     // omitted for sys
  text,
  time: '09:14',
  read: true                            // outgoing only
}
```

### Channel-type catalog (constant, top of script)
```js
CHANNEL_TYPES = {
  'tg-bot': { icon:'📨', label:'Telegram Bot' },
  'wa':     { icon:'💚', label:'WhatsApp Cloud' },
  'viber':  { icon:'💜', label:'Viber Bot' },
  'twilio': { icon:'💬', label:'Twilio SMS' },
  'fb':     { icon:'🔵', label:'Messenger' },
  'ig':     { icon:'📷', label:'Instagram Direct' },
}
```

### Status pill catalog
```js
STATUS_LABELS = {
  online:        { cls:'ok',    text:'● online' },
  offline:       { cls:'draft', text:'● offline' },
  auth_required: { cls:'warn',  text:'● auth_required' },
  error:         { cls:'err',   text:'● error' },
  draft:         { cls:'draft', text:'● draft' },
}
```

---

## 8. Function index

Grouped by concern. All are top-level functions in the single `<script>` block.

### Tab/navigation
- `setTab(tab)` — swaps screens, updates rail + tabbar active state, triggers screen-specific renders.

### Channels
- `renderChannels()` — re-renders the channel table, applies search/type/status filters, handles empty state.
- `toggleActive(id, on)` — flips the active checkbox, fires a toast.
- `openChannel(id)` / `reauth(id)` / `viewLogs(id)` — action button stubs.

### Wizard
- `openWizard()` — fresh wizard at step 1.
- `editChannel(id)` — opens wizard at step 3, prefilled.
- `resumeSetup(id)` — opens wizard at step 2 for draft channels.
- `closeWizard()`, `showWizardView(show)`
- `selectChannelType(el, type)`, `resetWizardSelection(type)`
- `goToStep(n)`, `nextStep()`, `prevStep()`
- `runConnectionTest()` — reveals the success block.
- `saveChannel()` — creates or updates a channel row, closes wizard, fires toast.

### Delete
- `askDelete(id)`, `closeDeleteModal()`, `confirmDelete()` — soft delete by default (sets `active=false`, `status='offline'`); checkbox for hard delete.

### Conversations
- `renderConversations()` — re-renders the list with all filters applied, then calls `renderActiveChat()` and `refreshUtilityBadge()`.
- `selectConversation(id)` — selects a conversation, marks unread=0.
- `renderActiveChat()` — re-renders the right pane.
- `renderMessage(m)` — single bubble HTML (handles text/image/file/sys).
- `sendMessage()` — appends outgoing bubble, fakes typing + read receipt.
- `convChannel(c)` — helper, returns the channel object linked to a conversation.

### Channel filter
- `renderChannelFilterDropdown()` — repopulates the per-channel select based on the active type chip.

### Lead page
- `renderLeadChat()` — re-renders the channel-switcher tabs and the active thread.
- `setLeadChannel(chId)` — switches active thread.
- `sendLeadMessage()` — appends to the active channel's thread, fakes read receipt.

### Utility bar / popup
- `toggleUtilPopup()`, `setUtilTab(tab)`
- `renderUtilPopup()` — renders the row list for the active tab + quick reply input.
- `utilOpenChat(convId)` — jumps to Conversations and selects.
- `utilOpenLead(convId)` — jumps to Lead Record.
- `utilQuickSend()` — sends to the first row in the current view.
- `refreshUtilityBadge()` — recomputes the unread badge.

### Toasts / utils
- `toast(variant, message)` — creates an SLDS toast, auto-dismisses.
- `escapeHtml(s)`, `initials(name)`, `formatTime(d)`

---

## 9. SLDS — what's real, what's a lookalike

**Real SLDS (uses CDN classes directly):**
- The toast stack (`slds-notify_toast`, `slds-theme_success`, `slds-theme_error`)
- The conversations search input (`slds-input`, `slds-form-element`, `slds-input-has-icon_left`)
- The All/Unread chips (`slds-button_outline-brand`, `slds-button_neutral`)
- The composer's attach/emoji/send buttons (`slds-button_icon-border`, `slds-button_brand`)

**Hand-rolled CSS that mimics SLDS** (everything else):
- `.panel`, `.panel-h`, `.panel-b` — card-like wrappers
- `.btn`, `.btn.brand`, `.btn.destructive`, `.btn.sm` — button system
- `.slds-tbl` — table styling
- `.pill` and variants — status pills
- `.wizard-steps`, `.wizard-step`, `.wizard-body`, `.wizard-foot` — wizard layout
- `.channel-card` — wizard step 1 cards
- `.modal-bg`, `.modal` — soft-delete modal
- `.lead-page`, `.lead-head`, `.lead-fields`, `.lead-chat`, `.lead-chat-source-tabs` — Lead screen
- `.mf-utility-bar`, `.mf-util-popup`, `.mf-util-row` — utility bar + popup
- `.mf-shell`, `.mf-rail`, `.mf-tabbar`, `.mf-screen` — overall shell
- `.mf-chat-list`, `.mf-conv-row`, `.mf-bubble`, `.mf-msg-area`, `.mf-composer`, `.mf-link-pill` — conversations layout

CSS variables at the top of `<style>` (`--slds-blue: #0176d3` etc.) are hand-picked hex values that mirror SLDS tokens by eye — they don't pull from the actual SLDS token system.

**Implication for the LWC build:** the prototype's CSS should not be copied verbatim. The next phase should replace nearly all custom classes with `lightning-*` base components and SLDS BEM utilities. See §11.

---

## 10. Seed data

### Channels (5 rows)
| id | name                  | type   | status         | active | msgs24 |
|----|-----------------------|--------|----------------|--------|--------|
| c1 | Sales Bot — EU        | tg-bot | online         | true   | 247    |
| c2 | Support WhatsApp      | wa     | online         | true   | 1,084  |
| c3 | Twilio SMS — US       | twilio | auth_required  | true   | 33     |
| c4 | Viber Marketing       | viber  | error          | false  | 0      |
| c5 | FB Page — MessageForge| fb     | draft          | false  | —      |

### Conversations (12 rows)
Each is linked to one of the channels by `channelId`. Maria Constantinou (cv1) is the demo conversation with the richest seed thread (text + image + system message).

### Lead threads
Three threads on the Maria lead: c1 (Telegram), c2 (WhatsApp), c3 (SMS), each 2–3 messages.

### Audit log
4 static rows for Recent Audit Events.

---

## 11. Mapping to the real LWC build

When this prototype becomes real Lightning Web Components, the structural mapping is:

| Prototype piece                          | Real component                                                    |
|------------------------------------------|-------------------------------------------------------------------|
| Channels list table                      | `messengerChannelList` LWC using `lightning-datatable`            |
| Wizard                                   | `channelSetupWizard` LWC using `lightning-path` + `lightning-input` |
| Channel-type cards (step 1)              | Custom radio-card group inside the wizard                         |
| Soft-delete modal                        | `lightning-modal` (or `lightning/modal` module)                   |
| Conversations list + chat                | `messengerChat` LWC, full mode                                    |
| Channel filter (chips + dropdown)        | Inside `messengerChat`, drives a `MessengerController` query     |
| Lead Record embedded chat                | Same `messengerChat` LWC, `mode='embedded'`, `@api recordId`     |
| Channel switcher tabs on Lead chat       | `lightning-tabset` inside the embedded mode                      |
| Bottom utility bar                       | Salesforce **Utility Bar**, configured in App Builder             |
| Messages popup contents                  | `messengerInbox` LWC registered as a utility item                |
| Toasts                                   | `lightning/platformShowToastEvent`                                |

The Apex contract referenced in the wizard legend:
```
listChannelTypes()
createApp(channelType, payload)
createChannel(appId, payload)
testConnection(channelId)
activateChannel(channelId)
registerWebhook(channelId)
```
Plus the chat side: `MessengerController.sendMessage`, `getChatList`, `getRecentMessages`, and Platform Event subscriptions to `tgint__Inbound_Message__e` / `tgint__Message_Delivery_Status__e` (per the original handoff doc §9).

---

## 12. Known gaps and TODOs

Things the prototype intentionally fakes or skips:

1. **No persistence.** Refresh = full reset.
2. **Wizard validation.** Fields are not validated; you can step through with empty inputs.
3. **Wizard is visually identical for all 6 channel types.** In the real build, step 2 (App Credentials) and step 3 (Channel Config) should branch by `wizard.type` — Telegram Bot needs API ID/Hash + Bot Token, WhatsApp needs WABA ID + Phone Number ID + Cloud API token, Twilio needs Account SID + Auth Token, etc.
4. **Edit reuses step 3 only.** A real edit flow may want to expose step 2 (rotate credentials) and step 4 (re-test).
5. **Resume Setup** jumps to step 2 unconditionally — should resume at the actual last completed step.
6. **Lead page is a single hardcoded Lead** (Maria Constantinou). All `Open lead` clicks route here.
7. **Cross-channel timeline merging** is not implemented — the Lead page shows separate tabs per channel. This is `[OPEN §10.1]` in the original handoff and needs a product decision.
8. **Optimistic outbound rendering** is faked with `setTimeout` for read receipts. The real flow is "insert pending → return Id → LWC shows immediately → Platform Event updates status".
9. **No WhatsApp template picker.** Required for outside-24h-window messages. Not in prototype.
10. **No reply-to threading** (`Reply_To_External_ID__c`).
11. **No edit indicators** (`Is_Edited__c`).
12. **No media-too-large placeholder** for the `media_too_large_phase2` error state.
13. **Settings screen** is empty.
14. **Audit log** is static seed data — does not update when actions happen.
15. **No keyboard navigation, no focus trap in modals, no ARIA beyond what SLDS provides by default.**
16. **No mobile layout testing.** The shell is desktop-first; the chat list at 320px will overflow on narrow viewports.

---

## 13. How to run

1. Open `index.html` in any modern browser (Chrome, Safari, Firefox).
2. No server, no build, no internet beyond the SLDS CDN fetch.
3. To test offline, download the SLDS CSS once and replace the CDN `<link>` with a local path.

---

## 14. How to extend (for the next agent)

**Adding a new screen:**
1. Add a button to the nav rail (`.mf-rail-btn[data-tab="..."]`).
2. Add a tab to the tabbar (`.mf-tab[data-tab="..."]`).
3. Add a `<section class="mf-screen" id="screen-...">` inside `.mf-workspace`.
4. Add a `display` toggle line inside `setTab()`.
5. Optionally call a render function from `setTab()` when the tab activates.

**Adding a new channel type:**
1. Add an entry to `CHANNEL_TYPES` at the top of the script.
2. Add a card in the wizard step 1 grid (`.channel-cards`).
3. Optionally add a chip in the conversations channel filter.
4. Add a filter option in the channels list type dropdown.

**Adding a new conversation:**
- Push a new object into `state.conversations` (or seed it inside `seedConversations()`). Make sure `channelId` references a real channel in `state.channels`. Then call `renderConversations()`.

**Adding a new utility bar item:**
- Add a `<button class="mf-utility-btn">` inside `.mf-utility-bar`. For a popup-style item, model it on the existing Messages popup and toggle it from a new state flag.

---

**Document owner:** prototype agent
**Companion file:** `index.html`
**Source spec:** `messageforge-phase1-technical-handoff.md` (the original Phase 1 architecture doc)
