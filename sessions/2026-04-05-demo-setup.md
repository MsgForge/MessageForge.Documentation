# Session: 2026-04-05 — Demo Setup (Telegram → Go → Salesforce)

## Goal
Get end-to-end demo running: Telegram message → Go backend (fly.io) → Salesforce Platform Event → Messenger_Message__c record.

## Infrastructure State

### fly.io Backend (`messageforge-backend`)
- **Region:** ams
- **Machines:** 2 (286d714fe75908 = active, 2876924f120738 = stopped/redundant)
- **Config changed:** `auto_stop_machines = "off"`, `auto_start_machines = false` — manual control only
- **Health:** https://messageforge-backend.fly.dev/health → `{"database":"healthy","status":"ok"}`

### fly.io Postgres (`messageforge-db`)
- **Machine:** d89702ec645ed8
- **Was stopped** at session start (role: error). Started manually.

### fly.io Secrets
| Secret | Value/Note |
|--------|-----------|
| DATABASE_URL | Set |
| WEBHOOK_SECRET | `0d7c6cd57e7edc867f4b095c8d3c7ed1fc110a26624d1895c9892b8eadccadf5` |
| ENCRYPTION_KEY | Set |
| SF_CLIENT_ID | Set (Consumer Key from "MessageForge Backend Dev" Connected App) |
| SF_INSTANCE_URL | `https://ghostwheel--tgdev.sandbox.my.salesforce.com` |
| SF_USERNAME | `van.surnin@profluenta.com.tgdev` |
| SF_PRIVATE_KEY | PEM key from `/tmp/messageforge-server.key` |
| SF_TOKEN_URL | `https://test.salesforce.com/services/oauth2/token` (NEW — added this session) |
| TELEGRAM_BOT_TOKEN | `8564822853:AAHP-Or33XQ9mPRkl6jad3xWtt5rqKPimDM` (NEW — added this session) |

### Salesforce Org
- **Org:** van.surnin@profluenta.com.tgdev (sandbox)
- **URL:** https://ghostwheel--tgdev.sandbox.my.salesforce.com
- **Namespace:** `tgint__` (but fields/objects accessed WITHOUT prefix in dev org)
- **Connected App:** "MessageForge Backend Dev" (created manually via UI — metadata deploy fails on sandbox)

### Telegram Bot
- **Token:** 8564822853:AAHP-Or33XQ9mPRkl6jad3xWtt5rqKPimDM
- **Webhook:** https://messageforge-backend.fly.dev/webhook/telegram
- **Secret token:** matches WEBHOOK_SECRET

## Manual Control Commands
```bash
# Start (DB first!)
fly machines start d89702ec645ed8 --app messageforge-db
fly machines start 286d714fe75908 --app messageforge-backend

# Stop (app first!)
fly machines stop 286d714fe75908 --app messageforge-backend
fly machines stop d89702ec645ed8 --app messageforge-db
```

## Issues Found & Fixed

### 1. Telegram Webhook Auth Mismatch (401 Unauthorized)
- **Problem:** Go backend used custom `X-Signature` HMAC-SHA256 middleware for all webhooks. Telegram sends `X-Telegram-Bot-Api-Secret-Token` as a plain string — incompatible.
- **Fix:** Added `TelegramSecretAuth()` middleware in `internal/server/middleware.go` that validates `X-Telegram-Bot-Api-Secret-Token` header. Added `RegisterTelegramWebhook()` method in `server.go`. Changed `app.go` to use `srv.RegisterTelegramWebhook()` for the Telegram route.
- **Files changed:**
  - `MessageForge.Backend/internal/server/middleware.go` — added `TelegramSecretAuth()`
  - `MessageForge.Backend/internal/server/server.go` — added `RegisterTelegramWebhook()`
  - `MessageForge.Backend/internal/messenger/app.go` — switched to `RegisterTelegramWebhook()`

### 2. JWT Token URL Wrong for Sandbox (app_not_found)
- **Problem:** Default token URL was `https://login.salesforce.com/services/oauth2/token`. Sandbox requires `https://test.salesforce.com/services/oauth2/token`.
- **Fix:** Added `SF_TOKEN_URL` env var to config. Wired it into `JWTAuthConfig` in `app.go`. Set fly secret.
- **Files changed:**
  - `MessageForge.Backend/internal/messenger/config/config.go` — added `SFTokenURL` field
  - `MessageForge.Backend/internal/messenger/app.go` — passes `cfg.SFTokenURL` to auth config

### 3. Namespace Prefix in Platform Event Name (404 Not Found)
- **Problem:** Platform Event names were hardcoded as `tgint__Inbound_Message__e` and `tgint__Message_Delivery_Status__e`. In the dev org, namespace prefix is NOT used.
- **Fix:** Added `SF_NAMESPACE` env var. If set, prepends `{namespace}__` to event names. Empty = no prefix (dev org).
- **Files changed:**
  - `MessageForge.Backend/internal/messenger/config/config.go` — added `SFNamespace` field
  - `MessageForge.Backend/internal/messenger/app.go` — dynamic event name with optional namespace prefix

### 4. Restricted Picklist Value (DML Error on Message Insert)
- **Problem:** Go backend sends `Protocol__c = "telegram"`, but `Messenger_Message__c.Protocol__c` picklist only had `mtproto` and `bot_api` values. Insert failed silently (caught by try/catch in trigger handler).
- **Symptom:** Chat was auto-created but Message was not saved. No visible error in UI.
- **Fix:** Added `telegram` picklist value to `Protocol__c` field definition.
- **Files changed:**
  - `MessageForge.Salesforce/force-app/main/default/objects/Messenger_Message__c/fields/Protocol__c.field-meta.xml` — added `<value><fullName>telegram</fullName>...`

### 5. fly.toml Auto-Stop (Machines kept stopping)
- **Problem:** `auto_stop_machines = "stop"` and `min_machines_running = 0` caused machines to stop between requests.
- **Fix:** Changed to `auto_stop_machines = "off"` and `auto_start_machines = false` for manual control.
- **Files changed:**
  - `MessageForge.Backend/fly.toml`

### 6. Developer Console Query Editor Bug
- **Problem:** `SELECT Id, Message_Text__c FROM Messenger_Message__c` fails in Developer Console Query Editor with column error, but works from SF CLI and Anonymous Apex.
- **Status:** UNRESOLVED. Likely Developer Console metadata cache issue. Workaround: use Anonymous Apex or SF CLI.

## Current Result
- **3 messages successfully delivered** from Telegram to Salesforce:
  - "12345" | Lev Shabalin | telegram | 15:26:58
  - "12344556" | Lev Shabalin | telegram | 15:27:04
  - "Hello" | Lev Shabalin | telegram | 15:27:08
- Chat auto-created: "Chat 450081254"
- Full flow working: Telegram → Go webhook → Platform Event → SF Trigger → Messenger_Message__c + Messenger_Chat__c

## Architecture Notes for Subscriber Orgs vs Dev Org
| Aspect | Dev/Scratch Org | Subscriber Org (AppExchange) |
|--------|----------------|------------------------------|
| Object names | `Messenger_Message__c` | `tgint__Messenger_Message__c` |
| Field names | `Message_Text__c` | `tgint__Message_Text__c` |
| Platform Events | `Inbound_Message__e` | `tgint__Inbound_Message__e` |
| SF_NAMESPACE env var | empty | `tgint` |
| Connected App | Created manually via UI | Auto-deployed with managed package |
| SF_TOKEN_URL | `test.salesforce.com` (sandbox) | `login.salesforce.com` (production) |
| Custom Metadata secrets | Set manually in org | Set by admin post-install |

## Files Modified (Not Yet Committed)
### MessageForge.Backend
- `fly.toml` — auto_stop off
- `internal/server/middleware.go` — added TelegramSecretAuth
- `internal/server/server.go` — added RegisterTelegramWebhook
- `internal/messenger/app.go` — TelegramWebhook + SF_TOKEN_URL + SF_NAMESPACE
- `internal/messenger/config/config.go` — SF_TOKEN_URL + SF_NAMESPACE

### MessageForge.Salesforce
- `force-app/main/default/objects/Messenger_Message__c/fields/Protocol__c.field-meta.xml` — added "telegram" picklist value
