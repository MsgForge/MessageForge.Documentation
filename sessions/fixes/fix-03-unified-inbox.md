# Fix #3: Unified Multi-Channel Chat Inbox

**Priority:** P1 — major UX gap
**Agent:** `salesforce-dev` + `architect`

---

## Current State

`messengerChat` filters chats by a single `channelId`: `WHERE Channel__c = :channelId`. An agent handling Telegram + WhatsApp sees only one channel at a time. No unified inbox exists.

## Architecture Decision

**Approach:** When `_channelId` is null/unset, load chats from ALL active channels. When set, filter by that channel. This is backwards-compatible — the existing single-channel behavior works when `channelId` is passed via flexipage property.

**Data model already supports this:** `Messenger_Chat__c` has `Channel__c` lookup with `Channel__r.Channel_Type_API_Name__c` accessible (confirmed — already used in `sendMessage()` at line 104).

---

## Implementation Plan

### Step 1: Add `getAllActiveChats()` Apex method

**File:** `MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls`

```apex
@AuraEnabled(cacheable=true)
public static List<Messenger_Chat__c> getAllActiveChats() {
    List<Messenger_Chat__c> chats = [
        SELECT Id, Chat_Title__c, Chat_Type__c, Chat_External_ID__c,
               Last_Message_At__c, Channel__c, Status__c,
               Last_Message_Preview__c, Unread_Count__c,
               Assigned_User__c, Contact__c, Lead__c,
               Contact__r.Name, Lead__r.Name,
               Channel__r.Channel_Type_API_Name__c,
               Channel__r.Display_Name__c
        FROM Messenger_Chat__c
        WHERE Channel__r.Active__c = true
        ORDER BY Last_Message_At__c DESC NULLS LAST
        LIMIT 200
    ];
    SObjectAccessDecision decision = Security.stripInaccessible(AccessType.READABLE, chats);
    return (List<Messenger_Chat__c>) decision.getRecords();
}
```

**Key additions vs `getChats()`:**
- No `channelId` filter — returns chats from ALL active channels
- Includes `Channel__r.Channel_Type_API_Name__c` for platform badge
- Includes `Channel__r.Display_Name__c` for channel name display
- Includes `Contact__r.Name` and `Lead__r.Name` for agent context

### Step 2: Update `getChats()` to also include relationship fields

**File:** `MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls`

Update existing `getChats()` query (line 11) to include the same relationship fields for consistency:

```apex
SELECT Id, Chat_Title__c, Chat_Type__c, Chat_External_ID__c,
       Last_Message_At__c, Channel__c, Status__c,
       Last_Message_Preview__c, Unread_Count__c,
       Assigned_User__c, Contact__c, Lead__c,
       Contact__r.Name, Lead__r.Name,
       Channel__r.Channel_Type_API_Name__c,
       Channel__r.Display_Name__c
FROM Messenger_Chat__c
WHERE Channel__c = :channelId
ORDER BY Last_Message_At__c DESC NULLS LAST
LIMIT 200
```

### Step 3: Import and wire `getAllActiveChats` in LWC

**File:** `MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js`

```javascript
import getAllActiveChats from '@salesforce/apex/MessengerController.getAllActiveChats';

// Replace the single wire with conditional logic:
// When _channelId is set → use getChats (existing)
// When _channelId is null → use getAllActiveChats (new unified view)

@wire(getChats, { channelId: '$_channelId' })
wiredChats;

@wire(getAllActiveChats)
wiredAllChats;

// Update filteredChats getter to use appropriate data source
get activeChatData() {
    if (this._channelId) {
        return this.wiredChats?.data || [];
    }
    return this.wiredAllChats?.data || [];
}
```

**Note:** The wire for `getAllActiveChats` has no parameters so it fires immediately on component load. The wire for `getChats` only fires when `_channelId` is set.

### Step 4: Add platform badge to chat list items

**File:** `MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.html`

Add a small platform indicator next to each chat title:

```html
<!-- Inside the chat list item, after chat title -->
<template lwc:if={chat.platformIcon}>
    <lightning-icon
        icon-name={chat.platformIcon}
        size="xx-small"
        class="slds-m-left_xx-small"
        alternative-text={chat.platformLabel}>
    </lightning-icon>
</template>
```

### Step 5: Add platform icon mapping to JS

```javascript
const PLATFORM_ICONS = {
    'Telegram_Bot': { icon: 'utility:chat', label: 'Telegram' },
    'Telegram_User': { icon: 'utility:chat', label: 'Telegram' },
    'WhatsApp_Cloud': { icon: 'utility:phone_portrait', label: 'WhatsApp' },
    'Viber_Bot': { icon: 'utility:comments', label: 'Viber' },
    'SMS_Twilio': { icon: 'utility:sms', label: 'SMS' }
};
```

### Step 6: Enrich `filteredChats` getter with platform data

Update the `filteredChats` getter (lines 438-462) to include platform info and contact/lead name:

```javascript
get filteredChats() {
    const chats = this.activeChatData;
    if (!chats) return [];

    return chats
        .filter(chat => {
            if (!this.searchTerm) return true;
            const term = this.searchTerm.toLowerCase();
            return (chat.Chat_Title__c || '').toLowerCase().includes(term)
                || (chat.Last_Message_Preview__c || '').toLowerCase().includes(term)
                || (chat.Contact__r?.Name || '').toLowerCase().includes(term)
                || (chat.Lead__r?.Name || '').toLowerCase().includes(term);
        })
        .map(chat => {
            const platformKey = chat.Channel__r?.Channel_Type_API_Name__c;
            const platformConfig = PLATFORM_ICONS[platformKey] || {};
            const contactName = chat.Contact__r?.Name || chat.Lead__r?.Name || null;

            return {
                ...chat,
                initials: this.getInitials(chat.Chat_Title__c),
                lastMessagePreview: chat.Last_Message_Preview__c || chat.Chat_Type__c || '',
                formattedTime: this.formatMessageTime(chat.Last_Message_At__c),
                hasUnread: chat.Unread_Count__c > 0,
                isSelected: chat.Id === this.selectedChatId,
                platformIcon: platformConfig.icon || null,
                platformLabel: platformConfig.label || platformKey || '',
                contactName: contactName,
                channelName: chat.Channel__r?.Display_Name__c || ''
            };
        });
}
```

### Step 7: Add optional channel filter dropdown

```html
<!-- At top of chat list sidebar, before search -->
<template lwc:if={showChannelFilter}>
    <lightning-combobox
        label="Channel"
        value={selectedChannelFilter}
        options={channelFilterOptions}
        onchange={handleChannelFilterChange}
        variant="label-hidden"
        class="slds-m-bottom_x-small">
    </lightning-combobox>
</template>
```

The filter options come from the unique channels in the loaded chats.

### Step 8: Update `autoSelectChannel()` for unified mode

When no channel is auto-selected (e.g., multiple active channels), stay in unified mode:

```javascript
autoSelectChannel() {
    getChannels()
        .then((channels) => {
            if (!channels || channels.length === 0) return;

            const activeChannels = channels.filter(ch => ch.Active__c && ch.Status__c === 'active');

            if (activeChannels.length === 1) {
                // Single active channel — select it directly
                this._channelId = activeChannels[0].Id;
                this.connectionStatus = 'online';
            } else if (activeChannels.length > 1) {
                // Multiple active channels — stay in unified mode (_channelId stays null)
                this.connectionStatus = 'online';
                this._multiChannel = true;
            } else {
                // No active channels
                this.connectionStatus = 'offline';
            }
        })
        .catch((error) => {
            console.error('[messengerChat] autoSelectChannel failed:', JSON.stringify(error));
        });
}
```

### Step 9: Add test coverage

**File:** `MessageForge.Salesforce/force-app/main/default/classes/MessengerControllerTest.cls`

Add tests for `getAllActiveChats()`:
- Test with chats across 2 channels, verify both returned
- Test ordering by Last_Message_At__c
- Test that archived channel's chats are excluded
- Verify Contact__r.Name and Channel__r fields are accessible

---

## Verification

```bash
cd MessageForge.Salesforce
sf project deploy start --source-dir force-app/main/default/classes/MessengerController.cls --target-org sandbox
sf project deploy start --source-dir force-app/main/default/classes/MessengerControllerTest.cls --target-org sandbox
sf project deploy start --source-dir force-app/main/default/lwc/messengerChat --target-org sandbox
sf apex run test --class-names MessengerControllerTest --target-org sandbox --wait 10
```

---

## Execution Prompt

```
@salesforce-dev Implement unified multi-channel chat inbox in messengerChat LWC.

Context: Currently messengerChat only shows chats from one channel (filtered by channelId). Agents handling multiple platforms need a unified inbox.

Implementation:

1. **Apex — MessengerController.cls:**
   - Add `getAllActiveChats()` method: query Messenger_Chat__c WHERE Channel__r.Active__c = true, include Channel__r.Channel_Type_API_Name__c, Channel__r.Display_Name__c, Contact__r.Name, Lead__r.Name. Order by Last_Message_At__c DESC. FLS via stripInaccessible.
   - Update existing `getChats()` to also include Channel__r.Channel_Type_API_Name__c, Channel__r.Display_Name__c, Contact__r.Name, Lead__r.Name in its SELECT.

2. **LWC JS — messengerChat.js:**
   - Import and wire `getAllActiveChats` (no params, fires immediately)
   - Add `activeChatData` getter: if `_channelId` is set → use `wiredChats.data`, else → use `wiredAllChats.data`
   - Update `filteredChats` getter to use `activeChatData` and enrich with platform icon/label, contact/lead name, channel name
   - Add PLATFORM_ICONS map: Telegram_Bot/Telegram_User → utility:chat, WhatsApp_Cloud → utility:phone_portrait, etc.
   - Update `autoSelectChannel()`: if multiple active channels, stay in unified mode (_channelId stays null, connectionStatus = 'online', _multiChannel = true). If single active channel, select it directly.
   - Search should also match Contact__r.Name and Lead__r.Name

3. **LWC HTML — messengerChat.html:**
   - Add platform icon (lightning-icon, xx-small) next to chat title in list items
   - Add contact/lead name as subtitle line below chat title when available
   - Add channel name subtitle when in unified mode (_multiChannel)

4. **Tests — MessengerControllerTest.cls:**
   - Add testGetAllActiveChats: create 2 channels with chats, verify all returned
   - Test archived channel exclusion
   - Test relationship field access

Files:
- MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls
- MessageForge.Salesforce/force-app/main/default/classes/MessengerControllerTest.cls
- MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js
- MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.html

Deploy all and run MessengerControllerTest.
```
