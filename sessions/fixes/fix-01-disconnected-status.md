# Fix #1: messengerChat Shows "Disconnected" Despite Active Channel

**Priority:** P0 — blocks demo
**Agent:** `salesforce-dev`

---

## Root Cause Analysis

Three independent bugs combine to keep `connectionStatus = 'offline'`:

### Bug A: `getChannelStatus()` is `cacheable=true` but returns stale data

`MessengerController.getChannelStatus()` (line 48) is `@AuraEnabled(cacheable=true)`. Cacheable methods are cached by the Lightning Data Service and **never re-fetched** on imperative calls unless `refreshApex()` is used. Since `loadChannelStatus()` calls it imperatively (line 85), the first call may cache `'offline'` and subsequent calls return the stale cached value.

### Bug B: `.catch(() => {})` swallows all errors silently

Both `autoSelectChannel()` (line 80) and `loadChannelStatus()` (line 91) have empty `.catch()` blocks. If `getChannels()` or `getChannelStatus()` throws (FLS stripping removes `Status__c`, method not found, wire error), the component silently stays offline with zero diagnostics.

### Bug C: Go backend does not publish `Session_Status__e` on channel connect

The `handleSessionStatusEvent()` handler (line 178) only fires when a `Session_Status__e` Platform Event arrives. The Go backend logs show successful connection but never publishes this event. So the event-driven status update path is dead.

---

## Fix Steps

### Step 1: Make `getChannelStatus()` non-cacheable for imperative use

**File:** `MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls`

```apex
// Line 48-49: Change from:
@AuraEnabled(cacheable=true)
public static String getChannelStatus(Id channelId) {

// To:
@AuraEnabled
public static String getChannelStatus(Id channelId) {
```

**Why:** Imperative calls to cacheable methods return stale data. Since this method is only called imperatively (not wired), removing `cacheable=true` ensures fresh data on every call.

### Step 2: Add error logging to `.catch()` blocks

**File:** `MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js`

```javascript
// Line 80: Replace empty catch in autoSelectChannel()
.catch((error) => {
    console.error('[messengerChat] autoSelectChannel failed:', JSON.stringify(error));
});

// Line 91: Replace empty catch in loadChannelStatus()
.catch((error) => {
    console.error('[messengerChat] loadChannelStatus failed:', JSON.stringify(error));
});
```

### Step 3: Add retry with fresh fetch after autoSelectChannel sets channelId

**File:** `MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js`

The `loadChannelStatus()` call inside `autoSelectChannel()` may execute before the wire adapter acknowledges `_channelId`. Add a small delay or use `setTimeout` to ensure the property has settled:

```javascript
// Lines 69-81: Replace autoSelectChannel()
autoSelectChannel() {
    getChannels()
        .then((channels) => {
            if (channels && channels.length > 0) {
                const active = channels.find(ch => ch.Active__c && ch.Status__c === 'active')
                    || channels.find(ch => ch.Active__c)
                    || channels[0];
                this._channelId = active.Id;
                // Use the channel status directly from the channel record
                // instead of a separate server call, since getChannels() already has Status__c
                if (active.Active__c && active.Status__c === 'active') {
                    this.connectionStatus = 'online';
                } else {
                    this.connectionStatus = 'offline';
                }
            }
        })
        .catch((error) => {
            console.error('[messengerChat] autoSelectChannel failed:', JSON.stringify(error));
        });
}
```

**Why:** `getChannels()` already returns `Status__c` and `Active__c` — no need for a second server call to `getChannelStatus()`. This eliminates the caching bug entirely for the auto-select path.

### Step 4: Fix `handleSessionStatusEvent` channel ID comparison

**File:** `MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js`

```javascript
// Line 184: Currently compares to `this.channelId` (the @api getter)
// But _channelId is set via autoSelectChannel, and the getter returns this._channelId
// This should work, but verify the comparison uses the backing field:
if (!eventChannelId || eventChannelId !== this._channelId) return;
```

### Step 5: Keep `loadChannelStatus()` for explicit channelId path

When `channelId` is passed via flexipage property, `autoSelectChannel()` is skipped and `loadChannelStatus()` runs directly. With Step 1 (non-cacheable), this path will now work correctly.

---

## Verification

```bash
# Deploy and test
cd MessageForge.Salesforce
sf project deploy start --source-dir force-app/main/default/classes/MessengerController.cls --target-org sandbox
sf project deploy start --source-dir force-app/main/default/lwc/messengerChat --target-org sandbox

# Verify Apex method returns 'online'
echo "System.debug(MessengerController.getChannelStatus('a0sFV0003kiyDbkYAE'));" | sf apex run --target-org sandbox

# Open Messenger Console in browser, check:
# 1. Console tab shows no JS errors
# 2. Header shows green "Connected" pill
# 3. Chat list populates
```

---

## Execution Prompt

```
@salesforce-dev Fix the "Disconnected" status bug in messengerChat LWC.

Context: The component always shows "Disconnected" even though the channel is active in the database. Three bugs:

1. `MessengerController.getChannelStatus()` (line 48) is `cacheable=true` but called imperatively — returns stale cached 'offline'. Remove `cacheable=true` since it's only used imperatively.

2. Empty `.catch(() => {})` blocks in `autoSelectChannel()` (line 80) and `loadChannelStatus()` (line 91) swallow errors. Add `console.error` logging.

3. `autoSelectChannel()` makes a redundant second call to `getChannelStatus()` when `getChannels()` already returns `Status__c` and `Active__c`. Refactor to derive connection status directly from the channel record returned by `getChannels()`:
   - If `active.Active__c && active.Status__c === 'active'` → set `this.connectionStatus = 'online'`
   - Otherwise → keep `'offline'`
   - This eliminates the caching bug for the auto-select path entirely.

Files:
- `MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls` — line 48, remove `cacheable=true`
- `MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js` — lines 69-92, refactor autoSelectChannel + add error logging

Do NOT change the wire adapter for getChats, the empApi subscriptions, or the HTML template. Only fix the status initialization flow.

After changes, deploy both files to sandbox and run the Apex anonymous test:
System.debug(MessengerController.getChannelStatus('a0sFV0003kiyDbkYAE'));
```
