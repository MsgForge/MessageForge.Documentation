# MVP Issues Analysis — 2026-04-05

## Issue #1: OAuth / Connected App Should Auto-Deploy

### Current State
- `MessageForge_Backend.connectedApp-meta.xml` **already exists** in the package at `force-app/main/default/connectedApps/`
- It includes X.509 certificate, OAuth scopes (Api, RefreshToken), IP enforcement, 90-day refresh token
- **BUT**: During the demo session, the Connected App was "created manually via UI" because "metadata deploy fails on sandbox"

### Root Cause
Connected Apps deploy fine with **2GP managed packages** to subscriber orgs. The sandbox issue was a one-time dev environment problem (sandboxes sometimes reject Connected App metadata deploys if the org has restrictions). In production AppExchange installs, Connected Apps **do auto-deploy** with the managed package.

### What's Actually Missing
1. **No post-install script** — There's no Apex `PostInstallScript` class to auto-assign permission sets, seed custom metadata defaults, or guide the admin through initial setup
2. **No Named Credential** — The package uses raw `HttpRequest` callouts with manual URL construction instead of Named Credentials (which would abstract the OAuth config)
3. **No setup instructions in-app** — After install, the admin has no guidance on what to configure

### Verdict
The Connected App **will** auto-deploy via AppExchange. The real gap is post-install automation and admin guidance.

---

## Issue #2: Custom Objects Not Visible to Other Users

### Current State
- 3 Permission Sets exist: `Messenger_Viewer`, `Messenger_Agent`, `Messenger_Admin`
- **BUT**: No permission sets are auto-assigned on install
- **No OWD (Organization-Wide Defaults)** configured in the package
- **No page layouts** exist — so even if users have object access, there's no standard UI to view records

### Root Cause
Permission sets exist in metadata but:
1. They're never **assigned** — no post-install script assigns them to the installing admin
2. **Tab visibility** is missing — no custom tabs exist for any Messenger objects, so they don't appear in the App Launcher
3. **No Lightning App** defined — no `MessageForge` application in the App Launcher that groups tabs together

### What's Missing
1. Custom **Tabs** for key objects (Messenger_Channel__c, Messenger_Chat__c, Messenger_Message__c)
2. A **Lightning App** (`MessageForge`) that groups these tabs + LWC pages
3. **Flexipages** (Lightning Record Pages) for each object with appropriate layout
4. **Post-install script** to assign `Messenger_Admin` to the installing user
5. Tab visibility in permission sets

### Verdict
Objects exist but are invisible because there are no tabs, no app, no flexipages, and no auto-assigned permissions.

---

## Issue #3: Missing LWC Auto-Setup Pages

### Current State
- 3 LWC components exist: `messengerChat`, `channelSetupWizard`, `messengerMediaViewer`
- **ZERO flexipages** (Lightning App Pages) exist
- **ZERO tabs** exist
- **ZERO Lightning Applications** exist

### Root Cause
LWC components are built but never placed into a navigable UI:
- `channelSetupWizard` is marked `lightning__AppPage` capable but has no App Page to live on
- `messengerChat` is marked for `RecordPage`, `AppPage`, `HomePage` but has no page definitions
- There's no way for users to navigate to these components

### What's Missing
1. **App Page** flexipage for `channelSetupWizard` (e.g., "Channel Setup")
2. **App Page** flexipage for `messengerChat` (e.g., "Messenger Console")
3. **Record Pages** for Messenger_Channel__c and Messenger_Chat__c with embedded LWC
4. **Custom Tabs** pointing to these flexipages
5. **Lightning App** (`MessageForge`) tying everything together

### Verdict
Components are built but have no pages, tabs, or app to make them accessible.

---

## Issue #4: Can't Setup Telegram Bot Through LWC Flow

### Current State
- `channelSetupWizard` LWC exists with 4 steps: Select Platform → Enter Credentials → Review & Save → Connect
- `ChannelSetupController` Apex exists with methods: `getChannelTypes()`, `getCredentialFields()`, `createChannelWithConfig()`, `connectChannel()`
- `Channel_Type__mdt` has `Telegram_Bot` record with config metadata

### Root Cause
The wizard LWC is **built and functional in code** but:
1. **No flexipage** hosts it — it's not accessible from the UI (see Issue #3)
2. The `connectChannel()` method makes a callout to the Go middleware, but the **Go endpoint for channel connection** may not be fully wired
3. **No Telegram-specific bot setup flow** exists — the wizard is generic (supports all channel types) but the Telegram Bot flow needs: enter token → Go validates via `getMe` → Go registers webhook → channel activates

### What's Missing
1. Flexipage + Tab to access the wizard (see Issue #3)
2. Verify Go middleware has `/api/channels/connect` endpoint that handles Telegram Bot token validation and webhook registration
3. The wizard's Step 4 ("Connect") needs to handle the async flow: call Go → wait for validation → display success/error

### Verdict
The wizard exists in code but is inaccessible. Once accessible, verify the Go↔SF channel connection flow works end-to-end.

---

## Issue #5: Missing Chat LWC with All Needed Windows

### Current State
- `messengerChat` LWC exists with: chat list, message list, message input, media viewer, empApi subscriptions, session status display
- It has: `getChats()`, `getMessages()`, `sendMessage()`, `sendMediaMessage()` wired to Apex controller
- `messengerMediaViewer` sub-component exists for image/video lightbox

### Root Cause
The chat component is **built and feature-rich** but:
1. **Not placed on any page** — no flexipage, no tab (see Issue #3)
2. **No split-pane layout** — the component renders chat list + message list but may need proper CSS for a messenger-style layout (sidebar + main area)
3. **No "Console" layout** — Salesforce has a Console Navigation style that works best for chat apps (split view, pinned tabs), but no Lightning App with Console navigation exists

### What's Missing
1. **Lightning Console App** (not standard navigation) for messenger-style split view
2. Flexipages placing `messengerChat` as the main console view
3. Utility bar items for quick access to active chats
4. Record pages for `Messenger_Chat__c` with embedded `messengerChat` component
5. Possibly a **Home Page** component showing recent chats/unread counts

### Verdict
The chat LWC is built with real functionality (empApi subscriptions, media, status). The gap is entirely in packaging/navigation — no app, no pages, no tabs.

---

## Summary of All Gaps

| Gap | Category | Severity |
|---|---|---|
| No Lightning App definition | Navigation | CRITICAL |
| No Custom Tabs | Navigation | CRITICAL |
| No Flexipages (App Pages, Record Pages) | Navigation | CRITICAL |
| No Post-Install Script | Automation | HIGH |
| No auto permission set assignment | Access | HIGH |
| No page layouts for objects | UX | MEDIUM |
| Go channel connection endpoint verification | Integration | MEDIUM |
| Console-style navigation for chat | UX | MEDIUM |

All 5 issues trace back to one root cause: **the package has code but no navigation/packaging metadata**.

---

## Fix Prompts

Below are the prompts to execute sequentially to fix all MVP gaps.

### Prompt 1: Lightning App + Tabs + Navigation

```
Create the MessageForge Lightning App and Custom Tabs for the Salesforce managed package.

Work in: MessageForge.Salesforce/force-app/main/default/

Create these metadata files:

1. **Lightning App** (`applications/MessageForge.app-meta.xml`):
   - Label: "MessageForge"
   - Navigation type: Console (for chat-style split view)
   - Branding: use "standard:live_chat" icon
   - Include tabs: Messenger_Channels, Messenger_Chats, Channel_Setup, Messenger_Console
   - Default tab: Messenger_Console
   - Assign to Messenger_Admin and Messenger_Agent permission sets

2. **Custom Tabs**:
   - `tabs/Messenger_Channel__c.tab-meta.xml` — Object tab for Messenger_Channel__c, icon "custom:custom62"
   - `tabs/Messenger_Chat__c.tab-meta.xml` — Object tab for Messenger_Chat__c, icon "custom:custom60"
   - `tabs/Channel_Setup.tab-meta.xml` — Web tab pointing to Channel_Setup flexipage, icon "custom:custom63"
   - `tabs/Messenger_Console.tab-meta.xml` — Web tab pointing to Messenger_Console flexipage, icon "standard:live_chat"

3. Update **Permission Sets** to include tab visibility:
   - Messenger_Admin: all 4 tabs visible + default on
   - Messenger_Agent: Messenger_Chats + Messenger_Console visible + default on
   - Messenger_Viewer: Messenger_Chats visible (read-only)

Do NOT modify any Apex or LWC code. Only create/edit metadata XML files.
```

### Prompt 2: Flexipages (App Pages + Record Pages)

```
Create Lightning Flexipages for the MessageForge package.

Work in: MessageForge.Salesforce/force-app/main/default/flexipages/

Create these flexipage metadata files:

1. **Messenger_Console.flexipage-meta.xml** (App Page):
   - Type: AppPage
   - Full-width single region layout
   - Contains: `tgint__messengerChat` LWC component (takes full width/height)
   - Label: "Messenger Console"

2. **Channel_Setup.flexipage-meta.xml** (App Page):
   - Type: AppPage
   - Single region layout
   - Contains: `tgint__channelSetupWizard` LWC component
   - Label: "Channel Setup"

3. **Messenger_Channel_Record_Page.flexipage-meta.xml** (Record Page):
   - Type: RecordPage
   - Object: Messenger_Channel__c
   - Two-column layout (66/33)
   - Left: Record Detail component + Related Lists
   - Right: Channel status card (field section with Status__c, Session_Status__c, Active__c, Last_Connected_At__c)

4. **Messenger_Chat_Record_Page.flexipage-meta.xml** (Record Page):
   - Type: RecordPage
   - Object: Messenger_Chat__c
   - Two-column layout (66/33)
   - Left: `tgint__messengerChat` LWC (configured with channelId from record)
   - Right: Record Detail with key fields (Chat_Title__c, Status__c, Channel__c, Assigned_User__c, Contact__c, Lead__c)

Do NOT modify any Apex or LWC code. Only create flexipage XML files.
```

### Prompt 3: Post-Install Script + Permission Auto-Assignment

```
Create a PostInstallScript Apex class for the MessageForge managed package.

Work in: MessageForge.Salesforce/force-app/main/default/classes/

Create:

1. **PostInstallScript.cls**:
   - Implements `InstallHandler` interface
   - `onInstall(InstallContext context)` method:
     a. Assign `Messenger_Admin` permission set to the installing user (`context.installerID()`)
     b. Seed default `Messenger_Settings__mdt` if not present (use Metadata API deploy or just log a message since CMT can't be inserted via DML)
     c. Log install event to `Channel_Audit_Log__c` with Event_Type__c = 'PACKAGE_INSTALLED', Source__c = 'salesforce'
   - Handle errors gracefully (don't let install fail)
   - Use `without sharing` (system context needed for permission set assignment)

2. **PostInstallScriptTest.cls**:
   - Test install, upgrade, and uninstall scenarios
   - Verify permission set assignment
   - Verify audit log creation
   - 90%+ coverage

3. Update `sfdx-project.json`:
   - Add `postInstallScript` property pointing to `PostInstallScript`

Follow all project patterns: bulkified, error handling, bind variables.
```

### Prompt 4: Page Layouts for Objects

```
Create standard Page Layouts for key MessageForge custom objects so records are viewable in the standard Salesforce UI.

Work in: MessageForge.Salesforce/force-app/main/default/layouts/

Create layouts for:

1. **Messenger_Channel__c-Channel Layout.layout-meta.xml**:
   - Sections: Channel Information (Display_Name__c, Channel_Type_API_Name__c, Status__c, Active__c), Connection Details (Session_Status__c, Go_Connection_Ref__c, Last_Connected_At__c, External_Account_ID__c), Security (Is_Compromised__c, Credential_Rotated_At__c)
   - Related Lists: Messenger_Chat__c, Channel_User_Access__c, Channel_Audit_Log__c

2. **Messenger_Chat__c-Chat Layout.layout-meta.xml**:
   - Sections: Chat Info (Chat_Title__c, Chat_Type__c, Status__c, Channel__c), Participants (Assigned_User__c, Contact__c, Lead__c), Activity (Last_Message_At__c, Last_Message_Preview__c, Unread_Count__c)
   - Related Lists: Messenger_Message__c

3. **Messenger_Message__c-Message Layout.layout-meta.xml**:
   - Sections: Message (Message_Text__c, Direction__c, Message_Type__c, Sender_Name__c, Sent_At__c), Delivery (Delivery_Status__c, Delivery_Error__c, Protocol__c), Media (Media_URL__c, Media_MIME_Type__c, Has_File__c, Attachment_Count__c)

Do NOT modify any Apex or LWC code. Only create layout XML files.
```

### Prompt 5: Verify & Wire Telegram Bot Setup Flow

```
Verify and fix the end-to-end Telegram Bot setup flow through the channelSetupWizard LWC.

Research first, then fix:

1. Read `MessageForge.Salesforce/force-app/main/default/classes/ChannelSetupController.cls` — understand what `connectChannel()` does
2. Read `MessageForge.Salesforce/force-app/main/default/lwc/channelSetupWizard/` — understand the full 4-step flow
3. Read `MessageForge.Backend/internal/` — find the channel connection/registration endpoint

Verify:
- Does the Go backend have an endpoint that accepts channel connection requests from SF?
- Does it validate Telegram bot tokens via Bot API `getMe`?
- Does it register the webhook with Telegram via `setWebhook`?
- Does the `connectChannel()` Apex method make the right callout?
- Does the wizard display connection success/failure correctly?

Fix any gaps in the SF↔Go channel setup flow. The flow should be:
1. Admin enters bot token in channelSetupWizard
2. SF saves channel + config records
3. SF calls Go middleware to validate token + register webhook
4. Go responds with success/error
5. Wizard shows result and activates channel

Only fix what's broken. Don't refactor working code.
```

### Prompt 6: Console Chat Layout Polish

```
Review and improve the messengerChat LWC for a proper messenger-style console experience.

Read all files in:
- MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/
- MessageForge.Salesforce/force-app/main/default/lwc/messengerMediaViewer/

Verify these work correctly:
1. **Split pane layout**: Chat list sidebar (left, ~300px) + Message area (right, fills remaining)
2. **Chat list**: Shows chat title, last message preview, unread count badge, timestamp
3. **Message area**: Messages grouped by date, sender avatar/name, timestamp, delivery status icons
4. **Message input**: Text area with send button, file attachment button
5. **Live updates**: empApi subscriptions for inbound messages, delivery status, session status
6. **Media**: Click to open messengerMediaViewer lightbox
7. **Responsive**: Works in Console App sidebar width

Fix CSS/HTML layout issues only. Don't restructure the JavaScript logic unless something is clearly broken.
The component should look and feel like a modern messenger (similar to Telegram Web or WhatsApp Web).
```

### Execution Order

| Order | Prompt | Dependency | Est. Complexity |
|---|---|---|---|
| 1 | Prompt 1: App + Tabs | None | Low (XML only) |
| 2 | Prompt 2: Flexipages | Prompt 1 (tabs reference pages) | Low (XML only) |
| 3 | Prompt 3: Post-Install Script | None (can parallel with 1-2) | Medium (Apex) |
| 4 | Prompt 4: Page Layouts | None (can parallel with 1-2) | Low (XML only) |
| 5 | Prompt 5: Bot Setup Flow | Prompts 1-2 (needs accessible wizard) | Medium (cross-repo) |
| 6 | Prompt 6: Chat Polish | Prompts 1-2 (needs accessible chat) | Medium (CSS/HTML) |

**Parallelizable**: Prompts 1+3+4 can run simultaneously. Then 2. Then 5+6 simultaneously.
