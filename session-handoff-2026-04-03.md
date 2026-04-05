# Session Handoff Document — 2026-04-03

## Purpose

This document captures the current state of the MessageForge Salesforce implementation after debugging session on 2026-04-03. It documents fixed issues, remaining work, and critical context for continuing MVP implementation.

---

## Executive Summary

The channel creation wizard is now **functional**. Three critical bugs were fixed:
1. `stripInaccessible()` returning copies instead of modifying originals (ID propagation bug)
2. Missing field-level security (FLS) configuration
3. Missing custom metadata configuration records

**Status:** Channel creation wizard works. Connection to Go middleware will fail until real configuration values are set.

---

## Fixed Issues

### Issue #1: "Required fields are missing: [Channel]" on Telegram_Bot_Config__c

**Location:** `ChannelSetupController.cls` lines 200-204 and 66-67

**Root Cause:**
```apex
SObjectAccessDecision decision = Security.stripInaccessible(AccessType.CREATABLE, new List<SObject>{channel});
insert decision.getRecords();  // Inserts a COPY, not the original
createPlatformConfig(..., channel.Id, ...);  // channel.Id is NULL!
```

`Security.stripInaccessible().getRecords()` returns **stripped copies**, not the originals. The original `channel` object never received an ID because the copy was inserted, not the original.

**Fix Applied:**
```apex
List<SObject> strippedChannels = decision.getRecords();
insert strippedChannels;
Id channelId = (Id) strippedChannels[0].get('Id');  // Get ID from inserted record
createPlatformConfig(..., channelId, ...);
```

**Files Changed:**
- `force-app/main/default/classes/ChannelSetupController.cls`

**Deployed:** Yes

---

### Issue #2: "Insufficient permissions: secure query included inaccessible field"

**Root Cause:** User lacked field-level security (FLS) on `Messenger_Channel__c.Display_Name__c` field. The `connectChannel` method uses `WITH SECURITY_ENFORCED` which requires FLS.

**Fix Applied:**
1. Added field permissions to `Messenger_Admin.permissionset-meta.xml`
2. Assigned `Messenger_Admin` permission set to the user

**Important:** Salesforce does NOT allow FLS configuration on required fields via permission sets. Only optional fields can have FLS modified.

**Fields with FLS in permission set:**
- `Messenger_Channel__c.Display_Name__c`
- `Messenger_Channel__c.Active__c`
- `Telegram_Bot_Config__c.Bot_Username__c`

**Fields that CANNOT have FLS modified (required fields):**
- `Messenger_Channel__c.Channel_Type_API_Name__c`
- `Messenger_Channel__c.Status__c`
- `Telegram_Bot_Config__c.Bot_Token__c`

**Files Changed:**
- `force-app/main/default/permissionsets/Messenger_Admin.permissionset-meta.xml`

**Deployed:** Yes

---

### Issue #3: "Messenger Settings not configured"

**Root Cause:** Missing custom metadata records required by `connectChannel()`:
- `Encryption_Key__mdt.HMAC_Secret` - for signing requests to Go middleware
- `Messenger_Settings__mdt.Default` - contains Go server URL

**Fix Applied:** Created placeholder custom metadata records.

**Files Created:**
- `force-app/main/default/customMetadata/Encryption_Key.HMAC_Secret.md-meta.xml`
- `force-app/main/default/customMetadata/Messenger_Settings.Default.md-meta.xml`

**Deployed:** Yes

**⚠️ ACTION REQUIRED:** Update placeholder values with real values in Salesforce Setup:
- `Go_Server_URL__c` → Your actual Go middleware URL (e.g., `https://messenger-backend.example.com`)
- `HMAC_Secret__c` → Secure shared secret (must match Go middleware config)

---

## Current System State

### Deployed and Working

| Component | Status | Notes |
|-----------|--------|-------|
| `ChannelSetupController.cls` | ✅ Fixed | ID propagation bug fixed |
| `Messenger_Admin` permission set | ✅ Deployed | FLS for optional fields |
| Custom metadata records | ✅ Deployed | Placeholder values need update |
| `Messenger_Channel__c` object | ✅ Deployed | All fields deployed |
| `Telegram_Bot_Config__c` object | ✅ Deployed | Existing in org |
| `Channel_Type__mdt` records | ✅ Deployed | 8 channel types configured |

### Known Limitations

1. **Go Middleware Not Connected:** The `connectChannel()` method will fail because:
   - `Go_Server_URL__c` contains placeholder: `https://your-go-server.example.com`
   - No actual Go middleware deployed yet

2. **Missing Fields in Org:** The org schema may be out of sync with local source. Fields deployed in this session:
   - `Messenger_Channel__c.Display_Name__c`
   - `Messenger_Channel__c.Active__c`
   - `Messenger_Channel__c.Session_Status__c`
   - `Messenger_Channel__c.Session_Error_Detail__c`
   - `Messenger_Channel__c.Session_Last_Heartbeat__c`
   - `Messenger_Channel__c.External_Account_ID__c`
   - `Messenger_Channel__c.Go_Connection_Ref__c`
   - `Messenger_Channel__c.Last_Connected_At__c`
   - `Messenger_Channel__c.Credential_Rotated_At__c`
   - `Messenger_Channel__c.Is_Compromised__c`

3. **Bot_Username__c Missing from Org:** The local schema has `Telegram_Bot_Config__c.Bot_Username__c` but it wasn't deployed. The field may not exist in the org.

---

## Key Files Reference

### Apex Classes
- `force-app/main/default/classes/ChannelSetupController.cls` - LWC backend for wizard
- `force-app/main/default/classes/ChannelSetupControllerTest.cls` - Tests (verify coverage)

### Objects
- `force-app/main/default/objects/Messenger_Channel__c/` - Base channel object
- `force-app/main/default/objects/Telegram_Bot_Config__c/` - Bot configuration
- `force-app/main/default/objects/Channel_Type__mdt/` - Channel type definitions

### Permission Sets
- `force-app/main/default/permissionsets/Messenger_Admin.permissionset-meta.xml` - Full access
- `force-app/main/default/permissionsets/Messenger_Agent.permissionset-meta.xml` - Agent access
- `force-app/main/default/permissionsets/Messenger_Viewer.permissionset-meta.xml` - Read-only

### Custom Metadata
- `force-app/main/default/customMetadata/Encryption_Key.HMAC_Secret.md-meta.xml`
- `force-app/main/default/customMetadata/Messenger_Settings.Default.md-meta.xml`
- `force-app/main/default/customMetadata/Channel_Type.*.md-meta.xml` (8 records)

### LWC Components
- `force-app/main/default/lwc/channelSetupWizard/` - Main wizard component

---

## Remaining MVP Work

### Phase 1: Foundation (Current)
- [x] Channel creation wizard
- [x] FLS configuration
- [x] Custom metadata setup
- [ ] Configure real Go middleware URL
- [ ] Test end-to-end channel creation + connection

### Phase 2: Inbound Messaging
- [ ] `MessengerInboundAPI` REST endpoint
- [ ] HMAC validation on webhooks
- [ ] Message ingestion flow
- [ ] Platform Event handlers

### Phase 3: Outbound Messaging
- [ ] Go middleware integration
- [ ] Outbound message queue
- [ ] Delivery status tracking

---

## Debug Commands Reference

### Verify Deployment
```bash
sf project retrieve start --metadata "ApexClass:ChannelSetupController" --target-org van.surnin@profluenta.com.tgdev --output-dir retrieved_code
```

### Check FLS
```apex
Boolean hasAccess = Schema.SObjectType.Messenger_Channel__c.fields.Display_Name__c.isAccessible();
System.debug('Display_Name__c accessible: ' + hasAccess);
```

### Check Custom Metadata
```apex
Messenger_Settings__mdt settings = Messenger_Settings__mdt.getInstance('Default');
System.debug('Go Server URL: ' + settings.Go_Server_URL__c);
```

### Test Controller Method
```apex
String credentialsJson = '{"Bot_Token__c":"test_token"}';
Id result = ChannelSetupController.createChannelWithConfig('Test', 'Test Display', 'Telegram_Bot', credentialsJson);
System.debug('Created channel: ' + result);
```

---

## Org Information

- **Target Org:** `van.surnin@profluenta.com.tgdev`
- **User ID:** `005a3000001E2vpAAC`
- **Profile ID:** `00ea3000000v2oHAAQ`
- **Permission Set Assigned:** `Messenger_Admin` (ID: `0PSFV0000UEuz1M4QR`)

---

## Critical Code Patterns

### FLS-Safe DML Pattern
```apex
// ALWAYS get ID from inserted stripped record
SObjectAccessDecision decision = Security.stripInaccessible(AccessType.CREATABLE, records);
List<SObject> stripped = decision.getRecords();
insert stripped;
Id insertedId = (Id) stripped[0].get('Id');  // NOT originalRecord.Id
```

### Required Fields Cannot Have FLS Modified
When adding field permissions to permission sets, exclude required fields. Salesforce will reject deployment with error:
> "You cannot deploy to a required field: ObjectName.FieldName__c"

### WITH SECURITY_ENFORCED Requires FLS
Any query using `WITH SECURITY_ENFORCED` will fail if the user lacks FLS on ANY field in the SELECT clause. Ensure:
1. Permission set has FLS for all queried fields
2. Or remove `WITH SECURITY_ENFORCED` and use manual FLS checks

---

## Session Commands Run

```bash
# Deployed fixes
sf project deploy start --source-dir force-app/main/default/classes/ChannelSetupController.cls
sf project deploy start --source-dir force-app/main/default/objects/Messenger_Channel__c
sf project deploy start --source-dir force-app/main/default/permissionsets/Messenger_Admin.permissionset-meta.xml
sf project deploy start --source-dir force-app/main/default/customMetadata

# Assigned permission set
sf apex run -f assign_perm.apex  # Assigned Messenger_Admin to current user

# Verified configuration
sf apex run -f verify_config.apex  # Confirmed custom metadata records exist
```

---

## Next Agent Instructions

1. **Read this document first** - It contains all context from the debugging session.

2. **Verify org sync** - Run `sf project deploy start --source-dir force-app --check-only` to ensure local source matches org.

3. **Update configuration** - Replace placeholder values in custom metadata:
   - Go to Setup → Custom Metadata Types
   - Manage "Messenger Settings" → Edit "Default" record
   - Set real `Go_Server_URL__c` value
   - Manage "Encryption Key" → Edit "HMAC_Secret" record
   - Set secure `Key_Value__c` value

4. **Test wizard** - Create a test channel through the LWC wizard.

5. **Continue MVP** - Refer to `MessageForge.Documentation/plans/mvp-implementation-plan.md` for next phase.

---

*Document created: 2026-04-03*
*Session: Debug channel creation wizard*
*Author: Claude Code debugging session*