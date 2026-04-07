# Fix #2: Channel List / Management View in Channel Setup

**Priority:** P2 — UX gap
**Agent:** `salesforce-dev`

---

## Current State

`channelSetupWizard` only shows a 4-step "create new channel" wizard. No way to see existing channels. Admin must use raw Salesforce list view on the Messenger Channels object tab.

## Architecture Decision

Add a **list-first** pattern to the existing `channelSetupWizard` component:
- New step 0 (`STEP_CHANNEL_LIST`) shown before the wizard
- Reuse `MessengerController.getChannels()` (already returns all fields needed)
- Channel cards with status indicators, platform badge, actions
- "Add New Channel" button transitions to existing step 1

**NOT creating a separate component** — keeps the setup flow unified in one place.

---

## Implementation Plan

### Step 1: Add `getExistingChannels()` to ChannelSetupController

**File:** `MessageForge.Salesforce/force-app/main/default/classes/ChannelSetupController.cls`

```apex
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

**Why a dedicated method instead of reusing MessengerController.getChannels():**
- Different ordering (active first, then by last connected)
- Controller separation: setup vs runtime
- Can evolve independently (add edit/delete capabilities later)

### Step 2: Add list state to channelSetupWizard.js

**File:** `MessageForge.Salesforce/force-app/main/default/lwc/channelSetupWizard/channelSetupWizard.js`

Add imports, properties, and handlers:
- Import `getExistingChannels` from `ChannelSetupController`
- Add `@wire(getExistingChannels) existingChannels`
- Add `showChannelList = true` state (default view)
- Add `handleNewChannel()` → sets `showChannelList = false`, enters wizard step 1
- Add `handleBackToList()` → resets wizard, sets `showChannelList = true`
- Add `handleArchiveChannel(event)` → calls `archiveChannel()` (already exists in controller)
- Add computed properties for channel card data enrichment (status config, platform icon)

### Step 3: Add channel list HTML section

**File:** `MessageForge.Salesforce/force-app/main/default/lwc/channelSetupWizard/channelSetupWizard.html`

Insert before the existing wizard content, conditionally rendered when `showChannelList`:

```html
<template lwc:if={showChannelList}>
    <div class="slds-m-bottom_medium">
        <div class="slds-grid slds-grid_align-spread slds-m-bottom_medium">
            <h2 class="slds-text-heading_medium">Your Channels</h2>
            <lightning-button
                label="Add New Channel"
                variant="brand"
                icon-name="utility:add"
                onclick={handleNewChannel}>
            </lightning-button>
        </div>

        <template lwc:if={existingChannels.data}>
            <template lwc:if={hasChannels}>
                <div class="slds-grid slds-wrap slds-gutters">
                    <template for:each={channelCards} for:item="channel">
                        <div key={channel.Id} class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2 slds-large-size_1-of-3 slds-m-bottom_small">
                            <lightning-card title={channel.displayName}>
                                <lightning-badge slot="actions" label={channel.statusLabel} class={channel.statusClass}></lightning-badge>
                                <div class="slds-p-horizontal_small">
                                    <p><strong>Platform:</strong> {channel.platformLabel}</p>
                                    <p><strong>Status:</strong> {channel.statusLabel}</p>
                                    <p lwc:if={channel.lastConnected}><strong>Last Connected:</strong> {channel.lastConnected}</p>
                                </div>
                                <div slot="footer">
                                    <lightning-button-group>
                                        <lightning-button label="Archive" data-id={channel.Id} onclick={handleArchiveChannel}></lightning-button>
                                    </lightning-button-group>
                                </div>
                            </lightning-card>
                        </div>
                    </template>
                </div>
            </template>
            <template lwc:else>
                <div class="slds-illustration slds-illustration_small slds-p-around_large slds-text-align_center">
                    <p class="slds-text-body_regular">No channels configured yet. Click "Add New Channel" to get started.</p>
                </div>
            </template>
        </template>
    </div>
</template>
```

### Step 4: Add platform label mapping

```javascript
const PLATFORM_LABELS = {
    'Telegram_Bot': 'Telegram Bot',
    'Telegram_User': 'Telegram User (MTProto)',
    'WhatsApp_Cloud': 'WhatsApp Cloud API',
    'WhatsApp_OnPrem': 'WhatsApp On-Premises',
    'Viber_Bot': 'Viber Bot',
    'SMS_Twilio': 'SMS (Twilio)'
};
```

### Step 5: Update `channelSetupWizard.js-meta.xml`

Update `headerTitle` default to "Channel Management" since the component now shows list + wizard.

### Step 6: Add test coverage

**File:** `MessageForge.Salesforce/force-app/main/default/classes/ChannelSetupControllerTest.cls`

Add test for `getExistingChannels()`:
- Test with 0 channels
- Test with multiple channels (active + archived)
- Verify ordering (active first)
- Verify FLS enforcement

---

## Verification

```bash
cd MessageForge.Salesforce
sf project deploy start --source-dir force-app/main/default/classes/ChannelSetupController.cls --target-org sandbox
sf project deploy start --source-dir force-app/main/default/classes/ChannelSetupControllerTest.cls --target-org sandbox
sf project deploy start --source-dir force-app/main/default/lwc/channelSetupWizard --target-org sandbox
sf apex run test --class-names ChannelSetupControllerTest --target-org sandbox --wait 10
```

---

## Execution Prompt

```
@salesforce-dev Add a channel list view to the channelSetupWizard LWC.

Context: The setup wizard only shows "create new channel" flow. Admins need to see existing channels before creating new ones.

Implementation:
1. Add `getExistingChannels()` method to `ChannelSetupController.cls` — query all Messenger_Channel__c ordered by Active__c DESC, Last_Connected_At__c DESC. Use Security.stripInaccessible() for FLS. Return Id, Name, Channel_Type_API_Name__c, Display_Name__c, Status__c, Session_Status__c, Active__c, Last_Connected_At__c, External_Account_ID__c, CreatedDate.

2. In `channelSetupWizard.js`:
   - Import and wire `getExistingChannels`
   - Add `showChannelList = true` state property (default view)
   - Add `handleNewChannel()` → hides list, shows wizard step 1
   - Add `handleBackToList()` → resets wizard state, shows list
   - Add `channelCards` getter that enriches channel data with platform labels and status config
   - Add `handleArchiveChannel(event)` → calls existing `archiveChannel()` method

3. In `channelSetupWizard.html`:
   - Wrap existing wizard content in `<template lwc:if={showWizard}>`
   - Add channel list section with `<template lwc:if={showChannelList}>` containing:
     - Header with "Your Channels" title + "Add New Channel" brand button
     - Grid of lightning-cards (3 columns on large, 2 on medium, 1 on small)
     - Each card shows: display name, platform type, status badge (colored), last connected time
     - Footer with Archive button
     - Empty state message when no channels exist

4. Add test in `ChannelSetupControllerTest.cls` for `getExistingChannels()`.

Platform label mapping: Telegram_Bot → "Telegram Bot", Telegram_User → "Telegram User (MTProto)", WhatsApp_Cloud → "WhatsApp Cloud API"

Files:
- MessageForge.Salesforce/force-app/main/default/classes/ChannelSetupController.cls
- MessageForge.Salesforce/force-app/main/default/classes/ChannelSetupControllerTest.cls
- MessageForge.Salesforce/force-app/main/default/lwc/channelSetupWizard/channelSetupWizard.js
- MessageForge.Salesforce/force-app/main/default/lwc/channelSetupWizard/channelSetupWizard.html
- MessageForge.Salesforce/force-app/main/default/lwc/channelSetupWizard/channelSetupWizard.css (optional, for card styling)

After changes, deploy all files and run ChannelSetupControllerTest.
```
