# Salesforce for Go Developers

## What Salesforce Actually Is

Think of Salesforce as a **hosted, multi-tenant application platform with a built-in database, ORM, REST API, UI framework, and deployment pipeline** — all in one. You don't provision servers. You don't manage a database. Everything runs on Salesforce's infrastructure.

**Go analogy:** Imagine if your Go app, its PostgreSQL database, its REST API, and its React frontend were all hosted on a single platform that enforced strict resource quotas per tenant and auto-scaled for you. That's Salesforce.

## Key Concepts Map

| Salesforce Term | Go/Backend Equivalent | Explanation |
|---|---|---|
| **Org** | A tenant / deployment | One customer's entire Salesforce instance. Own data, code, config |
| **Object** | A database table | `Account`, `Contact` are standard. Custom: `Telegram_Message__c` (`__c` = custom) |
| **Field** | A column | Custom fields also get `__c`: `Message_Text__c` |
| **Record** | A row | One instance of an object |
| **Apex** | Server-side Java-like code | Salesforce's proprietary language. Runs ON Salesforce servers, not yours |
| **SOQL** | SQL (simpler, restricted) | `SELECT Id, Name FROM Account WHERE Name = 'Acme'`. No JOINs — use relationship traversal |
| **Trigger** | Database trigger / event hook | Apex code that runs before/after record insert/update/delete |
| **LWC** | React/Vue components | JavaScript UI. HTML templates + JS controllers. Runs in user's browser |
| **Managed Package** | A distributable app | Bundle of objects, Apex, LWC distributed via AppExchange |
| **Namespace** | Go module path | Unique prefix (e.g., `tgint__`) prevents naming conflicts |
| **Governor Limits** | Resource quotas (ulimits) | Hard runtime limits per transaction. Exceed = unhandleable exception + rollback |
| **Platform Events** | Pub/sub bus (NATS, Kafka) | Internal event bus. External systems publish in, LWC subscribes real-time |
| **Flow** | No-code workflow builder | Drag-and-drop automation. Admins build these |
| **Connected App** | OAuth2 client registration | How your Go middleware authenticates to Salesforce APIs |
| **Protected Custom Metadata Type** | Encrypted config store (Vault) | Key-value config accessible only within managed package. For secrets |

## How Salesforce Authenticates Your Go Middleware

**OAuth 2.0 JWT Bearer Flow (recommended for server-to-server):**

1. Create a **Connected App** in Salesforce Setup
2. Generate an X.509 certificate and upload it
3. Go middleware creates a JWT signed with the private key
4. Exchange the JWT for an access token via `POST /services/oauth2/token`
5. Use the access token: `Authorization: Bearer <token>`
6. Handle token refresh — tokens expire

**Example Go -> Salesforce API call:**
```
POST https://<instance>.salesforce.com/services/data/v60.0/sobjects/Messenger_Message__c
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "Message_Text__c": "Hello from Telegram",
  "Chat__c": "a0B5f000001234567",
  "Direction__c": "inbound"
}
```

## Development Tools

| Tool | What It Does | Go Equivalent |
|---|---|---|
| **Salesforce CLI (sf)** | Deploy code, create scratch orgs, run tests | `go build` + `go test` + deploy |
| **VS Code + SF Extensions** | IDE for Apex + LWC | VS Code + Go extension |
| **Scratch Org** | Disposable temporary SF environment | Docker container for testing |
| **Sandbox** | Copy of production for staging/QA | Staging environment |
| **Developer Edition Org** | Free SF org for development | Free tier cloud account |
| **Package Version** | Snapshot of managed package, installable | Tagged release / Docker image |

## Salesforce Editions

| Edition | API Calls/24h | Platform Events | File Storage | Price |
|---|---|---|---|---|
| Essentials | 15,000 | Limited | 1 GB | ~$25/user/mo |
| Professional | 15,000 | Limited | 10 GB | ~$80/user/mo |
| Enterprise | 100,000+ | 250K/hr publish | 10 GB | ~$165/user/mo |
| Unlimited | Unlimited | 250K/hr publish | Unlimited | ~$330/user/mo |

**Our target:** Enterprise edition minimum. Professional is too constrained for active Telegram parsing.
