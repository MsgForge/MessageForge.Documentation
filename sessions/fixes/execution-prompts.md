# Execution Prompts — Step-by-Step Sessions

Each prompt below is designed to run in its own Claude Code session.
Copy-paste the prompt block into a new session. Each is self-contained with full context.

---

## Fix #1: messengerChat "Disconnected" Status (P0 — Demo Blocker)

### Session 1.1 — Apex: Remove cacheable from getChannelStatus

```
Fix the stale cache bug in MessengerController.getChannelStatus().

File: MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls

Problem: `getChannelStatus()` at line 48 is `@AuraEnabled(cacheable=true)` but it's called imperatively by the LWC (not wired). Cacheable methods return stale cached data on imperative calls. The method correctly returns 'online' when run via anonymous Apex, but the LWC gets a cached 'offline' value.

Change line 48 from:
  @AuraEnabled(cacheable=true)
To:
  @AuraEnabled

Do NOT change anything else in the file. Do NOT change getChats(), getMessages(), or getChannels() — those are wired and need cacheable.

After the edit, deploy to sandbox:
  cd MessageForge.Salesforce && sf project deploy start --source-dir force-app/main/default/classes/MessengerController.cls --target-org sandbox

Then verify the method returns 'online' for the active channel:
  echo "Messenger_Channel__c ch = [SELECT Id FROM Messenger_Channel__c WHERE Active__c = true AND Status__c = 'active' LIMIT 1]; System.debug(MessengerController.getChannelStatus(ch.Id));" | sf apex run --target-org sandbox
```

### Session 1.2 — LWC: Fix autoSelectChannel to derive status from getChannels response

```
Fix the messengerChat LWC initialization flow to derive connection status from the channel record instead of making a separate server call.

File: MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js

Current bug: `autoSelectChannel()` (lines 69-81) calls `getChannels()` which already returns `Status__c` and `Active__c` fields, then makes a redundant call to `getChannelStatus()` which suffered from a caching bug. The `.catch(() => {})` blocks at lines 80 and 91 also swallow all errors silently.

Make these exact changes:

1. Replace `autoSelectChannel()` (lines 69-81) with:

```javascript
autoSelectChannel() {
    getChannels()
        .then((channels) => {
            if (channels && channels.length > 0) {
                const active = channels.find(ch => ch.Active__c && ch.Status__c === 'active')
                    || channels.find(ch => ch.Active__c)
                    || channels[0];
                this._channelId = active.Id;
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

2. Replace the `.catch()` in `loadChannelStatus()` (line 91) with:

```javascript
.catch((error) => {
    console.error('[messengerChat] loadChannelStatus failed:', JSON.stringify(error));
});
```

Do NOT change anything else — don't touch the wire adapters, empApi subscriptions, HTML template, send handlers, or any other method.

After the edit, deploy:
  cd MessageForge.Salesforce && sf project deploy start --source-dir force-app/main/default/lwc/messengerChat --target-org sandbox
```

### Session 1.3 — Verify Fix #1

```
Verify that the messengerChat "Disconnected" status fix is deployed and working.

Run these checks in order:

1. Verify Apex method works:
  cd MessageForge.Salesforce
  echo "Messenger_Channel__c ch = [SELECT Id FROM Messenger_Channel__c WHERE Active__c = true AND Status__c = 'active' LIMIT 1]; System.debug('Channel: ' + ch.Id); System.debug('Status: ' + MessengerController.getChannelStatus(ch.Id));" | sf apex run --target-org sandbox

2. Verify getChannelStatus is no longer cacheable (should NOT have cacheable=true):
  Check MessengerController.cls line 48 — should be just `@AuraEnabled`

3. Verify autoSelectChannel derives status from getChannels response:
  Check messengerChat.js autoSelectChannel() — should set connectionStatus directly from active.Active__c && active.Status__c === 'active'

4. Verify error logging is in place:
  Check messengerChat.js — both .catch blocks should have console.error, not empty bodies

5. Run existing tests to confirm no regressions:
  cd MessageForge.Salesforce && sf apex run test --class-names MessengerControllerTest --target-org sandbox --wait 10

Report what you find. Do NOT make any changes.
```

---

## Fix #3: Unified Multi-Channel Inbox (P1 — Major UX Gap)

### Session 3.1 — Apex: Add getAllActiveChats method

```
Add a new `getAllActiveChats()` method to MessengerController that returns chats from ALL active channels.

File: MessageForge.Salesforce/force-app/main/default/classes/MessengerController.cls

Add this method after the existing `getChats()` method (after line 22, before `getMessages()`):

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

Also update the existing `getChats()` method's SOQL (lines 11-14) to include the same relationship fields for consistency. Add these fields to the SELECT:
  Contact__r.Name, Lead__r.Name, Channel__r.Channel_Type_API_Name__c, Channel__r.Display_Name__c

The updated getChats SELECT should be:
```
SELECT Id, Chat_Title__c, Chat_Type__c, Chat_External_ID__c,
       Last_Message_At__c, Channel__c, Status__c,
       Last_Message_Preview__c, Unread_Count__c,
       Assigned_User__c, Contact__c, Lead__c,
       Contact__r.Name, Lead__r.Name,
       Channel__r.Channel_Type_API_Name__c,
       Channel__r.Display_Name__c
```

Do NOT change any other methods.

After edits, deploy:
  cd MessageForge.Salesforce && sf project deploy start --source-dir force-app/main/default/classes/MessengerController.cls --target-org sandbox
```

### Session 3.2 — Apex Tests: Add getAllActiveChats test coverage

```
Add test methods for the new `getAllActiveChats()` method in MessengerController.

File: MessageForge.Salesforce/force-app/main/default/classes/MessengerControllerTest.cls

The existing @TestSetup (lines 28-59) creates one channel ('active' status) with one chat. This is sufficient for basic tests.

Add these test methods at the end of the class (before the closing `}`):

```apex
@isTest
static void testGetAllActiveChats() {
    Test.startTest();
    List<Messenger_Chat__c> chats = MessengerController.getAllActiveChats();
    Test.stopTest();

    System.assertEquals(1, chats.size(), 'Should have one chat from active channel');
    System.assertNotEquals(null, chats[0].Id);
    System.assertEquals('private', chats[0].Chat_Type__c);
}

@isTest
static void testGetAllActiveChatsExcludesArchivedChannels() {
    // Create an archived channel with a chat
    Messenger_Channel__c archivedCh = new Messenger_Channel__c(
        Name = 'Archived Channel',
        Channel_Type_API_Name__c = 'Telegram_Bot',
        Display_Name__c = 'Archived Bot',
        Status__c = 'archived',
        Active__c = false
    );
    insert archivedCh;

    Messenger_Chat__c archivedChat = new Messenger_Chat__c(
        Chat_External_ID__c = 'archived_chat_1',
        Chat_Title__c = 'Archived Chat',
        Chat_Type__c = 'private',
        Channel__c = archivedCh.Id,
        Last_Message_At__c = DateTime.now()
    );
    insert archivedChat;

    Test.startTest();
    List<Messenger_Chat__c> chats = MessengerController.getAllActiveChats();
    Test.stopTest();

    // Should only return chats from active channels, not archived
    for (Messenger_Chat__c chat : chats) {
        System.assertNotEquals(archivedCh.Id, chat.Channel__c,
            'Should not include chats from archived channel');
    }
}

@isTest
static void testGetAllActiveChatsMultipleChannels() {
    // Create a second active channel with a chat
    Messenger_Channel__c ch2 = new Messenger_Channel__c(
        Name = 'Second Channel',
        Channel_Type_API_Name__c = 'Telegram_Bot',
        Display_Name__c = 'Second Bot',
        Status__c = 'active',
        Active__c = true
    );
    insert ch2;

    Messenger_Chat__c chat2 = new Messenger_Chat__c(
        Chat_External_ID__c = 'chat_second_1',
        Chat_Title__c = 'Second Chat',
        Chat_Type__c = 'group',
        Channel__c = ch2.Id,
        Last_Message_At__c = DateTime.now()
    );
    insert chat2;

    Test.startTest();
    List<Messenger_Chat__c> chats = MessengerController.getAllActiveChats();
    Test.stopTest();

    System.assert(chats.size() >= 2, 'Should include chats from both active channels');
}

@isTest
static void testGetChatsIncludesRelationshipFields() {
    Messenger_Channel__c ch = [SELECT Id FROM Messenger_Channel__c LIMIT 1];

    Test.startTest();
    List<Messenger_Chat__c> chats = MessengerController.getChats(ch.Id);
    Test.stopTest();

    System.assertEquals(1, chats.size());
    // Relationship fields should be accessible (may be null but not throw)
    Messenger_Chat__c chat = chats[0];
    // Channel relationship should be populated
    System.assertNotEquals(null, chat.Channel__c);
}
```

After edits, deploy and run tests:
  cd MessageForge.Salesforce
  sf project deploy start --source-dir force-app/main/default/classes/MessengerControllerTest.cls --target-org sandbox
  sf apex run test --class-names MessengerControllerTest --target-org sandbox --wait 10

All tests must pass. Report results.
```

### Session 3.3 — LWC JS: Wire getAllActiveChats and add multi-channel support

```
Add unified multi-channel inbox support to the messengerChat LWC JavaScript.

File: MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js

Make these changes:

1. Add import at line 3 (after the existing getChats import):
```javascript
import getAllActiveChats from '@salesforce/apex/MessengerController.getAllActiveChats';
```

2. Add a new wire after line 55:
```javascript
@wire(getAllActiveChats)
allChats;
```

3. Add a tracked property after line 47 (after chatSearchTerm):
```javascript
@track _multiChannel = false;
```

4. Add a platform icon map constant after the SESSION_STATUS_CONFIG (after line 24):
```javascript
const PLATFORM_ICONS = {
    'Telegram_Bot': { icon: 'utility:chat', label: 'Telegram' },
    'Telegram_User': { icon: 'utility:chat', label: 'Telegram' },
    'WhatsApp_Cloud': { icon: 'utility:phone_portrait', label: 'WhatsApp' },
    'WhatsApp_OnPrem': { icon: 'utility:phone_portrait', label: 'WhatsApp' },
    'Viber_Bot': { icon: 'utility:comments', label: 'Viber' },
    'SMS_Twilio': { icon: 'utility:sms', label: 'SMS' }
};
```

5. Add a getter for the active chat data source (after the `filteredChats` getter, around line 462):
```javascript
get activeChatData() {
    if (this._channelId) {
        return this.chats?.data || [];
    }
    return this.allChats?.data || [];
}

get activeChatError() {
    if (this._channelId) {
        return this.chats?.error;
    }
    return this.allChats?.error;
}
```

6. Update the `filteredChats` getter (lines 438-462). Replace `const chats = this.chats.data || [];` at line 439 with:
```javascript
const chats = this.activeChatData;
```

Also update the `map()` callback inside `filteredChats` to enrich each chat with platform info and contact/lead name. Replace the return object in the map (lines 447-454) with:
```javascript
return {
    ...chat,
    initials,
    lastMessagePreview: lastMsg || chat.Chat_Type__c || '',
    hasUnread: unread > 0,
    formattedTime: lastTime ? this.formatChatTime(lastTime) : '',
    itemClass: chat.Id === this.selectedChatId ? 'chat-item-selected' : 'chat-item',
    platformIcon: platformConfig ? platformConfig.icon : null,
    platformLabel: platformConfig ? platformConfig.label : '',
    contactName: chat.Contact__r?.Name || chat.Lead__r?.Name || null,
    channelName: chat.Channel__r?.Display_Name__c || ''
};
```

And add this before the return:
```javascript
const platformKey = chat.Channel__r?.Channel_Type_API_Name__c;
const platformConfig = PLATFORM_ICONS[platformKey] || null;
```

Also extend the search filter (lines 457-461) to include contact/lead names:
```javascript
return enriched.filter(
    (c) =>
        (c.Chat_Title__c || '').toLowerCase().includes(term) ||
        (c.lastMessagePreview || '').toLowerCase().includes(term) ||
        (c.contactName || '').toLowerCase().includes(term)
);
```

7. Update `autoSelectChannel()` to support multi-channel mode. Replace the current implementation with:
```javascript
autoSelectChannel() {
    getChannels()
        .then((channels) => {
            if (!channels || channels.length === 0) return;

            const activeChannels = channels.filter(ch => ch.Active__c && ch.Status__c === 'active');

            if (activeChannels.length === 1) {
                this._channelId = activeChannels[0].Id;
                this.connectionStatus = 'online';
            } else if (activeChannels.length > 1) {
                // Multiple active channels — unified inbox mode
                this._channelId = null;
                this._multiChannel = true;
                this.connectionStatus = 'online';
            } else {
                const anyActive = channels.find(ch => ch.Active__c);
                if (anyActive) {
                    this._channelId = anyActive.Id;
                    this.connectionStatus = 'offline';
                }
            }
        })
        .catch((error) => {
            console.error('[messengerChat] autoSelectChannel failed:', JSON.stringify(error));
        });
}
```

8. Update `findSelectedChat()` (line 516-519) to search both data sources:
```javascript
findSelectedChat() {
    if (!this.selectedChatId) return null;
    const chats = this.activeChatData;
    return chats.find((c) => c.Id === this.selectedChatId) || null;
}
```

Do NOT change: empApi subscriptions, send/media handlers, message rendering, delivery status handling, or the HTML template.

After edits, deploy:
  cd MessageForge.Salesforce && sf project deploy start --source-dir force-app/main/default/lwc/messengerChat --target-org sandbox
```

### Session 3.4 — LWC HTML: Add platform badge and contact name to chat list

```
Add platform badge and contact/lead name to the messengerChat HTML chat list items.

File: MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.html

The JS now provides `platformIcon`, `platformLabel`, `contactName`, and `channelName` on each chat item in `filteredChats`.

Make these changes:

1. Update the chat data check (line 28). Change:
```html
<template if:true={chats.data}>
```
To:
```html
<template if:true={activeChatData}>
```

2. Update the error check (line 53). Change:
```html
<template if:true={chats.error}>
```
To:
```html
<template if:true={activeChatError}>
```

3. Add a platform icon next to the chat title. In the chat-item-top-row (line 39-42), add a platform icon after the title span. Change:
```html
<div class="chat-item-top-row">
    <span class="chat-item-title">{chat.Chat_Title__c}</span>
    <span class="chat-item-time">{chat.formattedTime}</span>
</div>
```
To:
```html
<div class="chat-item-top-row">
    <span class="chat-item-title">{chat.Chat_Title__c}</span>
    <template if:true={chat.platformIcon}>
        <lightning-icon icon-name={chat.platformIcon} size="xx-small" alternative-text={chat.platformLabel} class="platform-badge"></lightning-icon>
    </template>
    <span class="chat-item-time">{chat.formattedTime}</span>
</div>
```

4. Add contact/lead name as a subtitle. After the chat-item-bottom-row div (after line 48, before the closing </div> of chat-item-body), add:
```html
<template if:true={chat.contactName}>
    <div class="chat-item-contact-row">
        <span class="chat-item-contact">{chat.contactName}</span>
    </div>
</template>
```

Do NOT change: the message panel, send bar, media viewer, header, or any other section.

After edits, deploy:
  cd MessageForge.Salesforce && sf project deploy start --source-dir force-app/main/default/lwc/messengerChat --target-org sandbox
```

### Session 3.5 — LWC CSS: Add platform badge and contact name styles

```
Add CSS styles for the new platform badge and contact name in the chat list.

File: MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.css

Add these styles at the end of the file:

```css
/* Platform badge in chat list */
.platform-badge {
    margin-left: 4px;
    flex-shrink: 0;
}

/* Contact/Lead name row */
.chat-item-contact-row {
    display: flex;
    align-items: center;
    margin-top: 2px;
}

.chat-item-contact {
    font-size: 11px;
    color: var(--slds-g-color-neutral-base-60, #706e6b);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}
```

Do NOT change any existing styles.

After edits, deploy:
  cd MessageForge.Salesforce && sf project deploy start --source-dir force-app/main/default/lwc/messengerChat --target-org sandbox
```

---

## Fix #2: Channel List / Management View (P2 — Admin UX)

### Session 2.1 — Apex: Add getExistingChannels method

```
Add a `getExistingChannels()` method to ChannelSetupController for the channel list view.

File: MessageForge.Salesforce/force-app/main/default/classes/ChannelSetupController.cls

Add this method after the existing `getChannelTypes()` method (after line 30, before `createChannel()`):

```apex
/**
 * Returns all existing channels for the channel management list view.
 * Ordered: active channels first, then by last connected time.
 */
@AuraEnabled(cacheable=true)
public static List<Messenger_Channel__c> getExistingChannels() {
    List<Messenger_Channel__c> channels = [
        SELECT Id, Name, Channel_Type_API_Name__c, Display_Name__c,
               Status__c, Session_Status__c, Active__c,
               Last_Connected_At__c, External_Account_ID__c,
               CreatedDate
        FROM Messenger_Channel__c
        ORDER BY Active__c DESC, Last_Connected_At__c DESC NULLS LAST
        LIMIT 200
    ];
    SObjectAccessDecision decision = Security.stripInaccessible(AccessType.READABLE, channels);
    return (List<Messenger_Channel__c>) decision.getRecords();
}
```

Do NOT change any existing methods.

After the edit, deploy:
  cd MessageForge.Salesforce && sf project deploy start --source-dir force-app/main/default/classes/ChannelSetupController.cls --target-org sandbox
```

### Session 2.2 — Apex Tests: Add getExistingChannels test coverage

```
Add test methods for `getExistingChannels()` in ChannelSetupControllerTest.

File: MessageForge.Salesforce/force-app/main/default/classes/ChannelSetupControllerTest.cls

Add these test methods at the end of the class (before the closing `}`):

```apex
@isTest
static void testGetExistingChannelsEmpty() {
    // No channels created in this test — but @TestSetup in other methods may create data
    // Use a fresh context by deleting all channels first
    delete [SELECT Id FROM Messenger_Channel__c];

    Test.startTest();
    List<Messenger_Channel__c> channels = ChannelSetupController.getExistingChannels();
    Test.stopTest();

    System.assertEquals(0, channels.size(), 'Should return empty list');
}

@isTest
static void testGetExistingChannelsReturnsAll() {
    ChannelSetupController.testSkipFLS = true;

    Messenger_Channel__c ch1 = new Messenger_Channel__c(
        Name = 'Active Bot',
        Channel_Type_API_Name__c = 'Telegram_Bot',
        Display_Name__c = 'Active Bot',
        Status__c = 'active',
        Active__c = true,
        Last_Connected_At__c = DateTime.now()
    );
    Messenger_Channel__c ch2 = new Messenger_Channel__c(
        Name = 'Archived Bot',
        Channel_Type_API_Name__c = 'Telegram_Bot',
        Display_Name__c = 'Archived Bot',
        Status__c = 'archived',
        Active__c = false
    );
    insert new List<Messenger_Channel__c>{ch1, ch2};

    Test.startTest();
    List<Messenger_Channel__c> channels = ChannelSetupController.getExistingChannels();
    Test.stopTest();

    System.assert(channels.size() >= 2, 'Should return at least 2 channels');
    // Active channels should come first (ordered by Active__c DESC)
    Boolean foundActive = false;
    Boolean foundArchived = false;
    for (Messenger_Channel__c ch : channels) {
        if (ch.Name == 'Active Bot') foundActive = true;
        if (ch.Name == 'Archived Bot') foundArchived = true;
    }
    System.assert(foundActive, 'Should include active channel');
    System.assert(foundArchived, 'Should include archived channel');
}
```

After edits, deploy and run tests:
  cd MessageForge.Salesforce
  sf project deploy start --source-dir force-app/main/default/classes/ChannelSetupControllerTest.cls --target-org sandbox
  sf apex run test --class-names ChannelSetupControllerTest --target-org sandbox --wait 10

All tests must pass. Report results.
```

### Session 2.3 — LWC JS: Add channel list state and handlers

```
Add channel list view state management to channelSetupWizard.

File: MessageForge.Salesforce/force-app/main/default/lwc/channelSetupWizard/channelSetupWizard.js

Make these changes:

1. Add import at line 3 (after the existing imports):
```javascript
import getExistingChannels from '@salesforce/apex/ChannelSetupController.getExistingChannels';
import archiveChannel from '@salesforce/apex/ChannelSetupController.archiveChannel';
import { refreshApex } from '@salesforce/apex';
```

2. Add `wire` to the lwc import (line 1). Change:
```javascript
import { LightningElement, api, wire } from 'lwc';
```
(This is already correct — just verify `wire` is imported.)

3. Add state properties after line 14 (after `@api headerTitle`):
```javascript
// Channel list state
showChannelList = true;
isArchiving = false;
_wiredChannelsResult;
existingChannels = [];
```

4. Add wire for existing channels after the state properties:
```javascript
@wire(getExistingChannels)
wiredChannels(result) {
    this._wiredChannelsResult = result;
    if (result.data) {
        this.existingChannels = result.data;
    }
}
```

5. Add platform label map after the STEP constants (after line 11):
```javascript
const PLATFORM_LABELS = {
    'Telegram_Bot': 'Telegram Bot',
    'Telegram_User': 'Telegram (MTProto)',
    'WhatsApp_Cloud': 'WhatsApp',
    'Viber_Bot': 'Viber',
    'SMS_Twilio': 'SMS'
};

const STATUS_CONFIG = {
    'active': { label: 'Active', variant: 'success' },
    'draft': { label: 'Draft', variant: 'warning' },
    'archived': { label: 'Archived', variant: 'light' },
    'connecting': { label: 'Connecting', variant: 'warning' },
    'error': { label: 'Error', variant: 'error' }
};
```

6. Add computed properties for the channel list (add after the existing computed properties section):
```javascript
get hasExistingChannels() {
    return this.existingChannels.length > 0;
}

get showWizard() {
    return !this.showChannelList;
}

get channelCards() {
    return this.existingChannels.map(ch => {
        const statusCfg = STATUS_CONFIG[ch.Status__c] || STATUS_CONFIG.draft;
        return {
            ...ch,
            displayName: ch.Display_Name__c || ch.Name,
            platformLabel: PLATFORM_LABELS[ch.Channel_Type_API_Name__c] || ch.Channel_Type_API_Name__c,
            statusLabel: statusCfg.label,
            statusVariant: statusCfg.variant,
            lastConnected: ch.Last_Connected_At__c
                ? new Date(ch.Last_Connected_At__c).toLocaleString()
                : null,
            isActive: ch.Active__c
        };
    });
}
```

7. Add handlers for list/wizard navigation:
```javascript
handleNewChannel() {
    this.showChannelList = false;
    this.currentStep = STEP_SELECT_PLATFORM;
    this.resetWizardState();
}

handleBackToList() {
    this.showChannelList = true;
    this.resetWizardState();
    refreshApex(this._wiredChannelsResult);
}

handleArchiveChannel(event) {
    const channelId = event.currentTarget.dataset.id;
    if (!channelId) return;
    this.isArchiving = true;

    archiveChannel({ channelId })
        .then(() => {
            this.showToast('Success', 'Channel archived', 'success');
            return refreshApex(this._wiredChannelsResult);
        })
        .catch((error) => {
            this.showToast('Error', this.reduceError(error), 'error');
        })
        .finally(() => {
            this.isArchiving = false;
        });
}

resetWizardState() {
    this.currentStep = STEP_SELECT_PLATFORM;
    this.selectedChannelType = '';
    this.channelName = '';
    this.displayName = '';
    this.credentialFields = [];
    this.credentialValues = {};
    this.createdChannelId = null;
    this.connectSuccess = false;
    this.connectError = '';
}
```

8. Update `connectedCallback()` — remove `this.loadChannelTypes()` since channel types are only needed when entering the wizard. Move it to `handleNewChannel()`:
```javascript
connectedCallback() {
    // Channel types loaded on demand when wizard opens
}
```

And update `handleNewChannel()` to load types:
```javascript
handleNewChannel() {
    this.showChannelList = false;
    this.currentStep = STEP_SELECT_PLATFORM;
    this.resetWizardState();
    this.loadChannelTypes();
}
```

Do NOT change: saveChannel, handleConnect, validateStep2, or any existing wizard logic.

After edits, deploy:
  cd MessageForge.Salesforce && sf project deploy start --source-dir force-app/main/default/lwc/channelSetupWizard --target-org sandbox
```

### Session 2.4 — LWC HTML: Add channel list view template

```
Add the channel list view to channelSetupWizard HTML, shown before the wizard.

File: MessageForge.Salesforce/force-app/main/default/lwc/channelSetupWizard/channelSetupWizard.html

The JS now has: showChannelList (boolean), showWizard (boolean), hasExistingChannels, channelCards (array), handleNewChannel(), handleBackToList(), handleArchiveChannel().

Restructure the template:

1. Replace the entire content of the <template> tag with:

```html
<template>
    <!-- Channel List View -->
    <template lwc:if={showChannelList}>
        <lightning-card title="Channel Management" icon-name="standard:channel_programs">
            <lightning-button
                slot="actions"
                label="Add New Channel"
                variant="brand"
                icon-name="utility:add"
                onclick={handleNewChannel}>
            </lightning-button>

            <div class="slds-p-around_medium">
                <template lwc:if={hasExistingChannels}>
                    <div class="slds-grid slds-wrap slds-gutters">
                        <template for:each={channelCards} for:item="channel">
                            <div key={channel.Id} class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2 slds-large-size_1-of-3 slds-m-bottom_small">
                                <lightning-card title={channel.displayName}>
                                    <lightning-badge slot="actions" label={channel.statusLabel} class={channel.statusVariant}></lightning-badge>
                                    <div class="slds-p-horizontal_small slds-p-bottom_small">
                                        <dl class="slds-dl_horizontal slds-dl_horizontal_small">
                                            <dt class="slds-dl_horizontal__label"><p class="slds-truncate">Platform</p></dt>
                                            <dd class="slds-dl_horizontal__detail"><p>{channel.platformLabel}</p></dd>
                                            <dt class="slds-dl_horizontal__label"><p class="slds-truncate">Status</p></dt>
                                            <dd class="slds-dl_horizontal__detail"><p>{channel.statusLabel}</p></dd>
                                        </dl>
                                        <template lwc:if={channel.lastConnected}>
                                            <p class="slds-text-body_small slds-text-color_weak slds-m-top_xx-small">Last connected: {channel.lastConnected}</p>
                                        </template>
                                    </div>
                                    <div slot="footer">
                                        <template lwc:if={channel.isActive}>
                                            <lightning-button
                                                label="Archive"
                                                data-id={channel.Id}
                                                onclick={handleArchiveChannel}
                                                disabled={isArchiving}>
                                            </lightning-button>
                                        </template>
                                    </div>
                                </lightning-card>
                            </div>
                        </template>
                    </div>
                </template>
                <template lwc:else>
                    <div class="slds-align_absolute-center slds-p-around_large">
                        <div class="slds-text-align_center">
                            <p class="slds-text-heading_small slds-m-bottom_small">No channels configured</p>
                            <p class="slds-text-body_regular slds-text-color_weak">Click "Add New Channel" to set up your first messaging channel.</p>
                        </div>
                    </div>
                </template>
            </div>
        </lightning-card>
    </template>

    <!-- Wizard View (existing) -->
    <template lwc:if={showWizard}>
        <lightning-card title={headerTitle} icon-name="standard:channel_programs">

            <!-- Back to list link -->
            <lightning-button
                slot="actions"
                label="Back to Channels"
                variant="base"
                icon-name="utility:back"
                onclick={handleBackToList}>
            </lightning-button>

            <!-- Progress Indicator -->
            <lightning-progress-indicator current-step={currentStepValue} type="base" variant="base">
                <lightning-progress-step label="Select Platform" value="1"></lightning-progress-step>
                <lightning-progress-step label="Enter Credentials" value="2"></lightning-progress-step>
                <lightning-progress-step label="Review & Save" value="3"></lightning-progress-step>
                <lightning-progress-step label="Connect" value="4"></lightning-progress-step>
            </lightning-progress-indicator>

            <div class="slds-p-around_medium">

                <!-- Step 1: Select Platform -->
                <template lwc:if={isStep1}>
                    <div class="slds-text-heading_small slds-m-bottom_medium">
                        Select a messaging platform to connect
                    </div>

                    <template lwc:if={isLoadingTypes}>
                        <lightning-spinner alternative-text="Loading channel types" size="medium">
                        </lightning-spinner>
                    </template>

                    <template lwc:if={channelTypesError}>
                        <div class="slds-text-color_error slds-m-bottom_medium">
                            {channelTypesError}
                        </div>
                    </template>

                    <template lwc:if={hasChannelTypes}>
                        <lightning-radio-group
                            name="channelType"
                            label="Channel Type"
                            options={channelTypeOptions}
                            value={selectedChannelType}
                            onchange={handleChannelTypeChange}
                            required
                        ></lightning-radio-group>

                        <template lwc:if={selectedTypeDetail}>
                            <div class="slds-box slds-m-top_medium slds-theme_shade">
                                <p><strong>Messenger:</strong> {selectedTypeDetail.messenger}</p>
                                <p><strong>Protocol:</strong> {selectedTypeDetail.protocol}</p>
                                <p><strong>Supports Outbound:</strong> {selectedTypeDetail.supportsOutboundLabel}</p>
                            </div>
                        </template>
                    </template>
                </template>

                <!-- Step 2: Enter Credentials -->
                <template lwc:if={isStep2}>
                    <div class="slds-text-heading_small slds-m-bottom_medium">
                        Enter credentials for {selectedTypeLabel}
                    </div>

                    <lightning-input
                        label="Channel Name"
                        value={channelName}
                        onchange={handleChannelNameChange}
                        required
                        placeholder="e.g. My Support Bot"
                        class="slds-m-bottom_small"
                    ></lightning-input>

                    <lightning-input
                        label="Display Name"
                        value={displayName}
                        onchange={handleDisplayNameChange}
                        placeholder="Optional: user-facing name"
                        class="slds-m-bottom_small"
                    ></lightning-input>

                    <template lwc:if={isLoadingFields}>
                        <lightning-spinner alternative-text="Loading credential fields" size="small">
                        </lightning-spinner>
                    </template>

                    <template for:each={credentialFields} for:item="field">
                        <lightning-input
                            key={field.fieldName}
                            label={field.label}
                            type={field.inputType}
                            required={field.isRequired}
                            field-level-help={field.helpText}
                            data-field={field.fieldName}
                            value={field.value}
                            onchange={handleCredentialChange}
                            class="slds-m-bottom_small"
                        ></lightning-input>
                    </template>
                </template>

                <!-- Step 3: Review & Save -->
                <template lwc:if={isStep3}>
                    <div class="slds-text-heading_small slds-m-bottom_medium">
                        Review your channel configuration
                    </div>

                    <dl class="slds-dl_horizontal">
                        <dt class="slds-dl_horizontal__label">
                            <p class="slds-truncate">Channel Type</p>
                        </dt>
                        <dd class="slds-dl_horizontal__detail">
                            <p>{selectedTypeLabel}</p>
                        </dd>

                        <dt class="slds-dl_horizontal__label">
                            <p class="slds-truncate">Channel Name</p>
                        </dt>
                        <dd class="slds-dl_horizontal__detail">
                            <p>{channelName}</p>
                        </dd>

                        <dt class="slds-dl_horizontal__label">
                            <p class="slds-truncate">Display Name</p>
                        </dt>
                        <dd class="slds-dl_horizontal__detail">
                            <p>{displayNameOrDefault}</p>
                        </dd>

                        <template for:each={credentialSummary} for:item="cred">
                            <dt class="slds-dl_horizontal__label" key={cred.label}>
                                <p class="slds-truncate">{cred.label}</p>
                            </dt>
                            <dd class="slds-dl_horizontal__detail" key={cred.maskedValue}>
                                <p>{cred.maskedValue}</p>
                            </dd>
                        </template>
                    </dl>

                    <template lwc:if={isSaving}>
                        <lightning-spinner alternative-text="Saving channel" size="medium">
                        </lightning-spinner>
                    </template>
                </template>

                <!-- Step 4: Connect -->
                <template lwc:if={isStep4}>
                    <div class="slds-text-heading_small slds-m-bottom_medium">
                        Connect your channel to the messaging platform
                    </div>

                    <template lwc:if={createdChannelId}>
                        <div class="slds-box slds-m-bottom_medium slds-theme_success">
                            <p>Channel created successfully.</p>
                            <p>Channel ID: {createdChannelId}</p>
                        </div>

                        <template lwc:if={isConnecting}>
                            <lightning-spinner alternative-text="Connecting channel" size="medium">
                            </lightning-spinner>
                            <p class="slds-m-top_small">Initiating connection to the messaging platform...</p>
                        </template>

                        <template lwc:if={connectSuccess}>
                            <div class="slds-box slds-theme_success slds-m-top_medium">
                                <lightning-icon icon-name="utility:success" size="small" class="slds-m-right_x-small">
                                </lightning-icon>
                                Channel connected and activated. Bot token validated and webhook registered with Telegram.
                            </div>
                        </template>

                        <template lwc:if={connectError}>
                            <div class="slds-box slds-theme_error slds-m-top_medium">
                                <p>{connectError}</p>
                                <p class="slds-m-top_small">You can retry the connection or connect manually later.</p>
                            </div>
                        </template>

                        <template lwc:if={showConnectButton}>
                            <lightning-button
                                variant="brand"
                                label="Connect Channel"
                                onclick={handleConnect}
                                class="slds-m-top_medium"
                            ></lightning-button>
                        </template>
                    </template>
                </template>
            </div>

            <!-- Navigation Buttons -->
            <div slot="footer" class="slds-grid slds-grid_align-spread">
                <template lwc:if={showBackButton}>
                    <lightning-button
                        label="Back"
                        onclick={handleBack}
                        disabled={isProcessing}
                    ></lightning-button>
                </template>
                <template lwc:else>
                    <div></div>
                </template>

                <template lwc:if={showNextButton}>
                    <lightning-button
                        variant="brand"
                        label={nextButtonLabel}
                        onclick={handleNext}
                        disabled={isNextDisabled}
                    ></lightning-button>
                </template>
            </div>

        </lightning-card>
    </template>
</template>
```

After edits, deploy:
  cd MessageForge.Salesforce && sf project deploy start --source-dir force-app/main/default/lwc/channelSetupWizard --target-org sandbox
```

---

## Fix #5: Record Page FlexiPages (P3 — Cosmetic)

### Session 5.1 — Retrieve FlexiPages from Sandbox

```
Retrieve the correct FlexiPage metadata from the sandbox org to fix Record Page template names.

Run these commands:
  cd MessageForge.Salesforce
  sf project retrieve start --metadata FlexiPage --target-org sandbox

Then check what was retrieved:
  git diff force-app/main/default/flexipages/

Report the template <name> values found in:
- Messenger_Chat_Record_Page.flexipage-meta.xml
- Messenger_Channel_Record_Page.flexipage-meta.xml

If the record pages don't exist in the sandbox (retrieve returns nothing for them), report that they need to be created via Lightning App Builder first.

Verify the App Pages still have correct templates:
- Messenger_Console.flexipage-meta.xml should use flexipage:defaultAppHomeTemplate
- Channel_Setup.flexipage-meta.xml should use flexipage:defaultAppHomeTemplate

Do NOT make any manual edits to flexipage XML files.
```

---

## Fix #4: ConnectedApp Sandbox Documentation (P4 — Docs Only)

### Session 4.1 — Add Sandbox Setup Note

```
Add a note about Connected App sandbox limitations to the project documentation.

File to create: MessageForge.Documentation/reference/sandbox-setup-notes.md

Content:

# Sandbox Setup Notes

## Connected App — Manual Creation Required

The `MessageForge_Backend` Connected App is included in the managed package and deploys automatically to subscriber orgs via AppExchange install.

Some sandbox environments restrict Connected App creation via metadata deployment. If you see this error during `sf project deploy`:

> "You can't create a connected app. To enable connected app creation, contact Salesforce Customer Support."

### Workaround

Create the Connected App manually in your sandbox:

1. Go to **Setup > App Manager > New Connected App**
2. Configure:
   - **Name:** MessageForge Backend
   - **Contact Email:** your admin email
   - **Enable OAuth Settings:** checked
   - **Callback URL:** `https://login.salesforce.com/services/oauth2/callback`
   - **OAuth Scopes:** Full access (api), Perform requests at any time (refresh_token, offline_access)
   - **Use digital signatures:** Upload the certificate from `force-app/main/default/certs/` or generate a new self-signed cert
3. Save and note the Consumer Key for Go backend configuration

This is NOT required for production AppExchange installs.

## Record Pages — Create via App Builder

Record Pages (`Messenger_Chat_Record_Page`, `Messenger_Channel_Record_Page`) may fail to deploy due to template name mismatches across org editions.

### Workaround

Create record pages via Lightning App Builder UI, then retrieve:

```bash
sf project retrieve start --metadata FlexiPage --target-org sandbox
```

This captures the exact template name the org uses.

---

Only create this one file. Do NOT modify any other files.
```
