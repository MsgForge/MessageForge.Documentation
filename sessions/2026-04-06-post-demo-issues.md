# Post-Demo Issues — 2026-04-06

Issues discovered during MVP demo testing after completing all 6 fix prompts from the 2026-04-05 analysis.

---

## Issue #1: messengerChat LWC Shows "Disconnected" Despite Active Channel

### Severity: HIGH — blocks demo

### Current State
- Channel `a0sFV0003kiyDbkYAE` is `Status__c = 'active'`, `Active__c = true` in the database
- `getChannelStatus()` Apex method correctly returns `'online'` when called server-side
- `getChannels()` returns the active channel with all fields intact (verified via anonymous Apex)
- The Go backend log confirms: `"channel connected successfully", botUsername: "demo_poc_shaba_bot"`
- **But**: the messengerChat LWC header always shows the red "Disconnected" pill

### Root Cause (Partially Diagnosed)
The component initializes `connectionStatus = 'offline'` and relies on two mechanisms to update it:

1. **`autoSelectChannel()`** — calls `getChannels()` imperatively, finds the active channel, sets `this._channelId`, then calls `loadChannelStatus()` which calls `getChannelStatus()`. This flow works in Apex but the LWC never reflects the result.

2. **`Session_Status__e` Platform Events** — the component subscribes via empApi, but the Go backend does not publish this event during channel connect. So the event-driven path never fires either.

### What Was Already Tried
- Added `getChannelStatus()` Apex method that checks `Active__c && Status__c == 'active'` → returns `'online'`
- Added `autoSelectChannel()` in LWC `connectedCallback()` to auto-detect first active channel
- Refactored `@api channelId` into getter/setter backed by `@track _channelId` for wire reactivity
- Fixed channel selection to prefer `Status__c === 'active'` over just `Active__c`
- Deactivated stale draft channels so only one `Active__c = true` remains
- All Apex methods verified working via `sf apex run`
- Hard refresh, `?cache=false` attempted

### Likely Remaining Causes
1. **LWC bundle cache** — Salesforce aggressively caches LWC bundles. The deployed JS may not be the version running in the browser. Needs verification via browser DevTools (Sources tab → check the actual JS being executed).
2. **`autoSelectChannel()` silent failure** — The `.catch(() => {})` swallows all errors. If `getChannels()` throws (e.g., FLS issue for the running user, or the method isn't found), the component silently stays offline.
3. **Wire not re-firing** — Even with `@track _channelId`, the `@wire(getChats, { channelId: '$_channelId' })` may not re-evaluate if the property is set during `connectedCallback()` before the wire is initialized.
4. **Missing `channelId` on the FlexiPage** — The Messenger Console flexipage places `c:messengerChat` without passing any `channelId` attribute. The component must auto-detect it, which depends on `autoSelectChannel()` working.

### Files Involved
- `MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js` — lines 26-75 (init, auto-select, status load)
- `MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls` — `getChannelStatus()`, `getChannels()`
- `MessageForge.Salesforce/force-app/main/default/flexipages/Messenger_Console.flexipage-meta.xml` — no `channelId` property passed

### Recommended Fix Approach
1. Open browser DevTools → Console tab to check for JS errors when the component loads
2. Add `console.error` (temporarily) in the `.catch()` blocks to surface silent failures
3. Consider switching from imperative `getChannels()` to `@wire(getChannels)` for initial load — wire adapters are more reliable for initial data fetch
4. As a fallback: pass `channelId` explicitly on the flexipage via a Lightning App Builder component property, or use a parent wrapper component that resolves the channel first

---

## Issue #2: No Channel List / Management View in Channel Setup

### Severity: MEDIUM — UX gap

### Current State
- The `channelSetupWizard` LWC only shows a "create new channel" flow (4-step wizard)
- After creating a channel, there is no way to see existing channels from within the wizard
- The only way to view existing channels is via the Messenger Channels object tab (raw list view)
- There is no visual dashboard showing connected channels, their status, platform, and bot username

### Expected Behavior
When an admin opens the Channel Setup page, they should see:

1. **Channel list panel** — a table or card grid showing all existing channels with:
   - Display name / bot username
   - Platform (Telegram Bot, WhatsApp, etc.)
   - Connection status (connected/disconnected/error) with colored indicator
   - Last connected timestamp
   - Actions: Edit, Disconnect, Delete

2. **"Add Channel" button** — opens the existing 4-step wizard as a modal or navigates to step 1

3. **Multi-platform support** — the list should clearly show which platform each channel belongs to, since the system supports multiple messenger platforms (Telegram, WhatsApp, Viber, SMS per the architecture)

### Files Involved
- `MessageForge.Salesforce/force-app/main/default/lwc/channelSetupWizard/` — needs a list view added before the wizard steps
- `MessageForge.Salesforce/force-app/main/default/classes/ChannelSetupController.cls` — has `getChannelTypes()` but no `getExistingChannels()` method
- `MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls` — has `getChannels()` that could be reused

### Recommended Implementation
- Add a `channelList` state to the wizard: before step 1, show existing channels
- Reuse `MessengerController.getChannels()` or add a dedicated method to `ChannelSetupController`
- Show channel cards with status indicators (reuse the same status config as messengerChat)
- "Add New Channel" button transitions to the existing step 1

---

## Issue #3: messengerChat Should Show Chats Across All Channels

### Severity: MEDIUM — architectural UX gap

### Current State
- `messengerChat` LWC filters chats by a single `channelId`: `WHERE Channel__c = :channelId`
- The component can only show conversations from one channel at a time
- There is no way to see a unified inbox across multiple channels/platforms
- An agent handling both Telegram and WhatsApp would need to switch channels manually

### Expected Behavior
The Messenger Console should show a **unified chat list** combining conversations from all active channels:

1. **All-channel inbox** — chat list sidebar shows chats from all active channels, sorted by `Last_Message_At__c DESC`
2. **Platform indicator** — each chat item shows a small icon/badge indicating the source platform (Telegram, WhatsApp, etc.)
3. **Channel filter** (optional) — a dropdown or tab bar at the top of the chat list to filter by channel/platform
4. **Lead/Contact context** — each chat shows the linked Contact or Lead name, not just the chat title, so agents can identify who they're talking to across platforms

### Data Model Support
The data model already supports this:
- `Messenger_Chat__c` has `Channel__c` (lookup to `Messenger_Channel__c`)
- `Messenger_Channel__c` has `Channel_Type_API_Name__c` (platform identifier)
- `Messenger_Chat__c` has `Contact__c` and `Lead__c` lookups
- All chats across channels are in the same object — no schema changes needed

### Files Involved
- `MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js` — `@wire(getChats)` currently filters by single channel
- `MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.html` — chat list items need platform badge
- `MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls` — `getChats()` needs an all-channels variant

### Recommended Implementation
1. Add `getAllActiveChats()` Apex method: query chats from all active channels, include `Channel__r.Channel_Type_API_Name__c` and `Channel__r.Display_Name__c`
2. When `channelId` is null/undefined, use `getAllActiveChats()` instead of `getChats(channelId)`
3. Add platform icon to each chat list item (map `Channel_Type_API_Name__c` → icon)
4. Add optional channel filter dropdown in the chat list header
5. Show `Contact__r.Name` or `Lead__r.Name` as subtitle in chat items when available

---

## Issue #4: ConnectedApp Cannot Deploy to Sandbox

### Severity: LOW — sandbox-only, works in AppExchange

### Current State
- `MessageForge_Backend.connectedApp-meta.xml` exists in the package
- Deploying to sandbox returns: "You can't create a connected app. To enable connected app creation, contact Salesforce Customer Support."
- The Connected App was created manually in the sandbox via Setup UI during initial demo

### Root Cause
Some Salesforce sandboxes restrict Connected App creation via metadata deployment. This is an org-level setting, not a code issue.

### Impact
- Does NOT block AppExchange / 2GP managed package installs (Connected Apps deploy fine to subscriber orgs)
- Only affects sandbox development/testing environments
- Workaround: create the Connected App manually via Setup UI in sandbox

### No Code Change Needed
Document in setup guide that sandbox users may need to create the Connected App manually.

---

## Issue #5: Record Page FlexiPages Need Correct Template Name

### Severity: LOW — cosmetic, alternative exists

### Current State
- `Messenger_Channel_Record_Page.flexipage-meta.xml` and `Messenger_Chat_Record_Page.flexipage-meta.xml` exist but fail to deploy
- Multiple template names were tried: `flexipageBlankBackgroundTemplate`, `flexipage:RecordHomeTemplateDesktop`, `flexipage:defaultRecordHomeTemplate` — none valid
- The valid template name varies by Salesforce API version and org edition

### Workaround
Create record pages via Lightning App Builder UI, then retrieve:
```bash
sf project retrieve start --metadata FlexiPage --target-org sandbox
```
This captures the exact template name the org uses.

### Impact
The App Pages (Messenger Console, Channel Setup) deploy fine — only Record Pages are affected. Record pages are optional for MVP demo.

---

## Execution Priority

| Priority | Issue | Blocks Demo? | Effort |
|---|---|---|---|
| 1 | #1 — Disconnected status in messengerChat | YES | Small (likely JS debugging / cache issue) |
| 2 | #3 — Unified multi-channel chat inbox | No but major UX gap | Medium (Apex + LWC changes) |
| 3 | #2 — Channel list in setup wizard | No but confusing UX | Medium (new LWC section) |
| 4 | #5 — Record page flexipages | No | Small (retrieve from org) |
| 5 | #4 — ConnectedApp sandbox deploy | No | None (document only) |

---

## Agent Delegation Guide

| Issue | Recommended Agent | Why |
|---|---|---|
| #1 | `salesforce-dev` | LWC debugging, wire adapter behavior, empApi subscriptions |
| #2 | `salesforce-dev` | LWC component enhancement, Apex controller method |
| #3 | `salesforce-dev` + `architect` | Cross-channel query design, UI/UX for multi-platform inbox |
| #4 | None | Documentation only |
| #5 | `salesforce-dev` | Retrieve from org and commit |
