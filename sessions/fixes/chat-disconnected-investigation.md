# Bug: Messenger Console Always Shows "Disconnected"

## Status: RESOLVED (ADR-22)

## Symptom

After successfully connecting a channel via Channel Setup Wizard (screenshot confirms: "Channel connected and activated. Bot token validated and webhook registered with Telegram"), the Messenger Console still shows "Disconnected" status. Hard refresh doesn't fix it.

## Confirmed Facts

1. **Channel connection succeeds** — `connectChannel()` returns success, sets `Status__c = 'active'` and `Active__c = true` via direct DML (ChannelSetupController.cls:354-357)
2. **Go middleware is running** on `https://messageforge-backend.fly.dev`
3. **Settings configured** — `Messenger_Settings__mdt.Default` has Go_Server_URL, `Encryption_Key__mdt.HMAC_Secret` has HMAC key, Remote Site Setting is active
4. **Code is deployed** — all classes, LWC, flexipages deployed to sandbox as of 2026-04-06
5. **`getChannels()` is no longer cacheable** — `cacheable=true` was removed (was a suspected cause, but removing it didn't fix the issue)

## The Connection Status Flow

### Entry Point
`Messenger_Console.flexipage-meta.xml` renders `messengerChat` component **without passing channelId** (no component properties set).

### Component Initialization (`messengerChat.js`)

```
connectedCallback() (line 71)
  → _channelId is null (not passed from flexipage)
  → calls autoSelectChannel() (line 79)
```

### autoSelectChannel() (lines 83-109)

```javascript
autoSelectChannel() {
    getChannels()  // Apex call → MessengerController.getChannels()
        .then((channels) => {
            // Filter for channels where BOTH Active__c AND Status__c === 'active'
            const activeChannels = channels.filter(
                ch => ch.Active__c && ch.Status__c === 'active'
            );
            
            if (activeChannels.length === 1) {
                this._channelId = activeChannels[0].Id;
                this.connectionStatus = 'online';      // ← THIS is the only path to "Connected"
            } else if (activeChannels.length > 1) {
                this._multiChannel = true;
                this.connectionStatus = 'online';
            } else {
                // Falls through here → stays 'offline' → "Disconnected"
                const anyActive = channels.find(ch => ch.Active__c);
                if (anyActive) {
                    this._channelId = anyActive.Id;
                    this.connectionStatus = 'offline';  // ← Explicitly set to offline
                }
            }
        });
}
```

### Apex: getChannels() (MessengerController.cls:88-100)

```apex
@AuraEnabled
public static List<Messenger_Channel__c> getChannels() {
    List<Messenger_Channel__c> channels = [
        SELECT Id, Name, Channel_Type_API_Name__c, Display_Name__c,
               Status__c, Session_Status__c, External_Account_ID__c,
               Active__c, Last_Connected_At__c
        FROM Messenger_Channel__c
        ORDER BY Name ASC LIMIT 200
    ];
    SObjectAccessDecision decision = Security.stripInaccessible(AccessType.READABLE, channels);
    return (List<Messenger_Channel__c>) decision.getRecords();
}
```

## Possible Root Causes (Investigate In Order)

### 1. `stripInaccessible` Stripping `Active__c` Field

**Priority: HIGH — Most Likely Cause**

`Active__c` is a Checkbox field that is **NOT required** (`force-app/main/default/objects/Messenger_Channel__c/fields/Active__c.field-meta.xml`). It depends on FLS grants via permission sets.

`Active__c` IS in `Messenger_Admin` permission set (readable+editable), but:
- Is the user's profile a System Admin? System Admin should have all FLS by default.
- Does the user actually have `Messenger_Admin` permission set assigned?
- If `Active__c` is stripped, `ch.Active__c` becomes `undefined` (falsy) in JS → filter returns empty → "Disconnected"

**How to verify:**
```apex
// Run in Developer Console → Execute Anonymous
List<Messenger_Channel__c> channels = [
    SELECT Id, Name, Status__c, Active__c FROM Messenger_Channel__c
];
System.debug('Raw channels: ' + JSON.serializePretty(channels));

SObjectAccessDecision decision = Security.stripInaccessible(AccessType.READABLE, channels);
List<Messenger_Channel__c> stripped = (List<Messenger_Channel__c>) decision.getRecords();
System.debug('Stripped channels: ' + JSON.serializePretty(stripped));
System.debug('Removed fields: ' + decision.getRemovedFields());
```

If `getRemovedFields()` shows `Active__c` or `Status__c`, that's the bug.

**Fix:** Either:
- Assign `Messenger_Admin` permission set to the user
- OR remove `stripInaccessible` from `getChannels()` and use `WITH SECURITY_ENFORCED` instead (fails loudly instead of silently stripping)

### 2. Database Values Not Actually Updated

**Priority: MEDIUM**

Maybe `connectChannel()` DML didn't persist. Could be a trigger/workflow/flow reverting the values.

**How to verify:**
```apex
// Run in Developer Console → Execute Anonymous
List<Messenger_Channel__c> channels = [
    SELECT Id, Name, Status__c, Active__c, Last_Connected_At__c
    FROM Messenger_Channel__c
];
for (Messenger_Channel__c ch : channels) {
    System.debug(ch.Name + ' → Status: ' + ch.Status__c + ', Active: ' + ch.Active__c);
}
```

If `Status__c` is `'draft'` and `Active__c` is `false`, the DML in `connectChannel` failed silently or was reverted.

### 3. JS Comparison Issue

**Priority: LOW**

`ch.Status__c === 'active'` — strict equality. If the picklist value stored is `'Active'` (capital A) instead of `'active'`, this comparison fails.

**How to verify:** Check the picklist values in the field definition:
- `Status__c.field-meta.xml` defines value as `<fullName>active</fullName>` (lowercase)
- `connectChannel` sets `channel.Status__c = 'active'` (lowercase)
- These should match, but verify actual DB value via anonymous Apex above.

### 4. `getChannels()` Returning Empty Array

**Priority: LOW**

If CRUD check fails (user can't read `Messenger_Channel__c` at all), query returns empty → `autoSelectChannel` returns early at line 86.

**How to verify:** Check browser console for `[messengerChat] autoSelectChannel failed:` error message.

## Files Involved

| File | Lines | Role |
|------|-------|------|
| `force-app/main/default/lwc/messengerChat/messengerChat.js` | 71-109 | `connectedCallback` → `autoSelectChannel` → sets connectionStatus |
| `force-app/main/default/classes/MessengerController.cls` | 88-100 | `getChannels()` — queries channels + `stripInaccessible` |
| `force-app/main/default/classes/MessengerController.cls` | 70-86 | `getChannelStatus()` — only called when channelId is pre-set (NOT the case here) |
| `force-app/main/default/classes/ChannelSetupController.cls` | 354-357 | `connectChannel()` — sets Status__c='active', Active__c=true |
| `force-app/main/default/flexipages/Messenger_Console.flexipage-meta.xml` | 4-8 | Renders `messengerChat` WITHOUT channelId prop |
| `force-app/main/default/permissionsets/Messenger_Admin.permissionset-meta.xml` | 12-16 | Grants Active__c read/edit |
| `force-app/main/default/objects/Messenger_Channel__c/fields/Active__c.field-meta.xml` | all | Checkbox, NOT required, defaultValue=true |
| `force-app/main/default/objects/Messenger_Channel__c/fields/Status__c.field-meta.xml` | all | Required picklist: draft/connecting/active/suspended/disconnected/archived |

## What Was Already Tried

1. ~~Removed `cacheable=true` from `getChannels()`~~ — Didn't fix it
2. ~~Verified settings (CMDT, Remote Site Settings)~~ — All correct
3. ~~Deployed all code to sandbox~~ — Deployed successfully
4. ~~Hard refresh browser~~ — Still disconnected

## Recommended Debug Approach

1. **Open browser DevTools console** on the Messenger Console page → look for error messages from `autoSelectChannel`
2. **Run the anonymous Apex script** from Root Cause #1 above → check if `stripInaccessible` strips fields
3. **Run the anonymous Apex script** from Root Cause #2 above → verify actual DB values
4. Based on findings:
   - If fields stripped → fix permission sets or remove `stripInaccessible` from `getChannels()`
   - If DB values wrong → investigate `connectChannel` DML and any triggers/flows on `Messenger_Channel__c`
   - If both look correct → add `console.log` to `autoSelectChannel` to see what data the JS receives
