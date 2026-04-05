# Salesforce Deployment Errors — 2026-04-03

## Deployment Status: FAILED (52 errors)

Deploy ID: `0AfFV000cmyQB000AG`
Target Org: `van.surnin@profluenta.com.tgdev` (sandbox)

---

## Error Categories

### 1. Missing Sharing Models (11 objects)

Custom objects require `sharingModel` in their metadata:

| Object | Error |
|--------|-------|
| `Apple_Messages_Config__c` | Must specify a sharing model value |
| `Bird_Config__c` | Must specify a sharing model value |
| `Channel_User_Access__c` | Must specify a sharing model value |
| `Messenger_Attachment__c` | Must specify a sharing model value |
| `Messenger_Message__c` | Must specify a sharing model value |
| `Telegram_Bot_Config__c` | Must specify a sharing model value |
| `Telegram_User_Config__c` | Must specify a sharing model value |
| `Twilio_Config__c` | Must specify a sharing model value |
| `Viber_Config__c` | Must specify a sharing model value |
| `Vonage_Config__c` | Must specify a sharing model value |
| `WhatsApp_Config__c` | Must specify a sharing model value |

**Fix:** Add `<sharingModel>ReadWrite</sharingModel>` (or appropriate value) to each object's `.object-meta.xml` file.

---

### 2. Field Length Violations (7 fields)

Fields exceed maximum length limits:

| Object | Field | Max Allowed | Error Line |
|--------|-------|-------------|------------|
| `Bird_App__c` | `Access_Key__c` | 175 | 12:13 |
| `Messenger_Template__c` | `Template_Footer__c` | 255 | 173:13 |
| `Messenger_Template__c` | `Template_Header__c` | 255 | 182:13 |
| `Telegram_App__c` | `API_Hash__c` | 175 | 12:13 |
| `Twilio_App__c` | `Auth_Token__c` | 175 | 22:13 |
| `Vonage_App__c` | `API_Secret__c` | 175 | 22:13 |
| `WhatsApp_App__c` | `App_Secret__c` | 175 | 22:13 |

**Fix:** Reduce `<length>` in `.field-meta.xml` files to at or below the max.

---

### 3. Missing Fields Referenced in Apex

| Apex Class | Line | Missing Field |
|------------|------|---------------|
| `MessengerController.cls` | 31:16 | `Delivery_Error__c` on `Messenger_Message__c` |
| `MessengerControllerTest.cls` | 194:46, 201:34 | `Delivery_Error__c` |
| `MessengerOutboundQueueable.cls` | 44:51, 58:28, 67:51, 77:24 | `Delivery_Error__c` |
| `MessengerOutboundQueueableTest.cls` | 99:40, 101:38 | `Delivery_Error__c` |

**Fix:** Either:
- Create `Delivery_Error__c` field on `Messenger_Message__c`, OR
- Remove references to `Delivery_Error__c` from Apex classes

---

### 4. Missing Custom Objects Referenced in Code

| Apex Class | Missing Object |
|------------|----------------|
| `ChannelAccessService.cls` | `Channel_User_Access__c` |
| `ChannelAccessServiceTest.cls` | `Channel_User_Access__c` |
| `ChannelAccessTrigger.trigger` | `Channel_User_Access__c` |
| `ChannelSetupControllerTest.cls` | `Telegram_Bot_Config__c`, `Telegram_User_Config__c` |
| PermissionSet `Messenger_Admin` | `Telegram_Bot_Config__c` |
| PermissionSet `Messenger_Agent` | `Messenger_Attachment__c` |
| PermissionSet `Messenger_Viewer` | `Messenger_Attachment__c` |

**Fix:** These objects exist in the local metadata but weren't deployed. Ensure all custom objects are included in the deployment.

---

## Files to Fix

### Object Metadata (add sharingModel)
```
force-app/main/default/objects/Apple_Messages_Config__c/Apple_Messages_Config__c.object-meta.xml
force-app/main/default/objects/Bird_Config__c/Bird_Config__c.object-meta.xml
force-app/main/default/objects/Channel_User_Access__c/Channel_User_Access__c.object-meta.xml
force-app/main/default/objects/Messenger_Attachment__c/Messenger_Attachment__c.object-meta.xml
force-app/main/default/objects/Messenger_Message__c/Messenger_Message__c.object-meta.xml
force-app/main/default/objects/Telegram_Bot_Config__c/Telegram_Bot_Config__c.object-meta.xml
force-app/main/default/objects/Telegram_User_Config__c/Telegram_User_Config__c.object-meta.xml
force-app/main/default/objects/Twilio_Config__c/Twilio_Config__c.object-meta.xml
force-app/main/default/objects/Viber_Config__c/Viber_Config__c.object-meta.xml
force-app/main/default/objects/Vonage_Config__c/Vonage_Config__c.object-meta.xml
force-app/main/default/objects/WhatsApp_Config__c/WhatsApp_Config__c.object-meta.xml
```

### Field Metadata (reduce length)
```
force-app/main/default/objects/Bird_App__c/fields/Access_Key__c.field-meta.xml
force-app/main/default/objects/Messenger_Template__c/fields/Template_Footer__c.field-meta.xml
force-app/main/default/objects/Messenger_Template__c/fields/Template_Header__c.field-meta.xml
force-app/main/default/objects/Telegram_App__c/fields/API_Hash__c.field-meta.xml
force-app/main/default/objects/Twilio_App__c/fields/Auth_Token__c.field-meta.xml
force-app/main/default/objects/Vonage_App__c/fields/API_Secret__c.field-meta.xml
force-app/main/default/objects/WhatsApp_App__c/fields/App_Secret__c.field-meta.xml
```

### Apex Classes (fix missing field references)
```
force-app/main/default/classes/MessengerController.cls
force-app/main/default/classes/MessengerControllerTest.cls
force-app/main/default/classes/MessengerOutboundQueueable.cls
force-app/main/default/classes/MessengerOutboundQueueableTest.cls
```

---

## Resolution Priority

1. **HIGH** — Fix sharing models (blocks all objects)
2. **HIGH** — Fix field lengths (blocks deployment)
3. **MEDIUM** — Add missing `Delivery_Error__c` field OR remove references
4. **MEDIUM** — Verify all objects are included in deployment package