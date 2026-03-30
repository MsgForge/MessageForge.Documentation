# MessageForge Fix Register — 2026-03-30

**Source:** Parallel recheck review (6 agents) — [2026-03-30-recheck-review.md](./2026-03-30-recheck-review.md)
**Prior reviews:** [Architecture Review](./2026-03-29-architecture-review.md) | [Risk Register](./2026-03-29-risk-register.md) | [Security Findings](./security-findings.md)
**Scope:** All issues remaining after March 30 recheck — verified against current codebase

---

## Status Legend

| Label | Meaning |
|---|---|
| OPEN | Not started |
| IN PROGRESS | Work underway |
| FIXED | Verified in code |
| WONTFIX | Accepted risk with justification |

---

## Tier 0 — Blocks Integration Testing

These issues prevent any end-to-end message flow from working. Fix before any integration test attempt.

---

### FIX-001: Outbound JSON Contract Mismatch (SF -> Go)

| Field | Value |
|---|---|
| **Severity** | CRITICAL |
| **Status** | OPEN |
| **Component** | Go Backend + Salesforce |
| **Discovered** | 2026-03-30 (Architecture agent) |

**Problem:** Salesforce `MessengerOutboundService` serializes outbound messages using Apex default JSON (camelCase). The Go `outboundAPIRequest` struct expects snake_case JSON tags. Every outbound message from Salesforce will fail deserialization — all fields arrive as zero values.

**Go struct (expects snake_case):**

```
File: MessageForge.Backend/internal/messenger/outbound_handler.go:16-22

type outboundAPIRequest struct {
    ChatID      string `json:"chat_id"`
    MessageText string `json:"message_text"`
    MediaFileID string `json:"media_file_id,omitempty"`
    SFMessageID string `json:"sf_message_id"`
    Platform    string `json:"platform"`
}
```

**Apex class (sends camelCase):**

```
File: MessageForge.Salesforce/force-app/main/default/classes/MessengerOutboundService.cls:7-14

public class OutboundMessage {
    public String chatExternalId;
    public String text;
    public String messageType;
    public String mediaUrl;
    public String protocol;
    public String sfMessageExternalId;
}
```

**Field mapping mismatch:**

| Go JSON Tag | Apex Field Name | Match? |
|---|---|---|
| `chat_id` | `chatExternalId` | NO |
| `message_text` | `text` | NO |
| `media_file_id` | `mediaUrl` | NO |
| `sf_message_id` | `sfMessageExternalId` | NO |
| `platform` | `protocol` | NO |

**Fix options:**

- **Option A (recommended):** Change Go JSON tags to match Apex output:
  ```go
  type outboundAPIRequest struct {
      ChatExternalID      string `json:"chatExternalId"`
      Text                string `json:"text"`
      MessageType         string `json:"messageType"`
      MediaURL            string `json:"mediaUrl,omitempty"`
      Protocol            string `json:"protocol"`
      SFMessageExternalID string `json:"sfMessageExternalId"`
  }
  ```
- **Option B:** Build an explicit `Map<String,String>` in Apex with snake_case keys instead of serializing the class directly.

**Affected flows:** All outbound messages (agent sends from Salesforce LWC to Telegram).

**Test:** Send a message from the LWC chat component. Verify Go receives non-empty fields.

---

### FIX-002: SessionStatusTriggerHandler References Non-Existent Field

| Field | Value |
|---|---|
| **Severity** | CRITICAL |
| **Status** | OPEN |
| **Component** | Salesforce |
| **Discovered** | 2026-03-30 (Salesforce safety agent) |

**Problem:** `SessionStatusTriggerHandler` references `Channel_SF_ID__c` on the `Session_Status__e` Platform Event, but the actual field is named `Connection_SF_ID__c`. All session status events are silently dropped because the field value is always null.

**Current code (wrong field name):**

```
File: MessageForge.Salesforce/force-app/main/default/classes/SessionStatusTriggerHandler.cls:19,24-25

if (String.isBlank(evt.Channel_SF_ID__c)) {    // <-- wrong field name
    continue;
}
Id channelId;
try {
    channelId = Id.valueOf(evt.Channel_SF_ID__c);  // <-- wrong field name
```

**Correct field metadata:**

```
File: MessageForge.Salesforce/force-app/main/default/objects/Session_Status__e/fields/Connection_SF_ID__c.field-meta.xml
```

**Fix:** Replace all occurrences of `Channel_SF_ID__c` with `Connection_SF_ID__c` in `SessionStatusTriggerHandler.cls`.

**Lines to change:** 19, 24, 25 (and any other references in the file).

**Test:** Publish a `Session_Status__e` event with a valid `Connection_SF_ID__c`. Verify the handler processes it instead of skipping.

---

### FIX-003: OutboundWorker Never Wired — Messages Stuck Forever

| Field | Value |
|---|---|
| **Severity** | CRITICAL |
| **Status** | OPEN |
| **Component** | Go Backend |
| **Discovered** | 2026-03-30 (Architecture agent) |

**Problem:** The `OutboundWorker` is never instantiated with a concrete messenger adapter. The field is always `nil`. Messages enqueued via `POST /api/outbound` sit in the `outbound_queue` table with status `pending` indefinitely — no error, no warning logged at startup.

**Where it's left nil:**

```
File: MessageForge.Backend/internal/messenger/app.go:256-260

// Outbound worker needs an adapter that can send messages.
// For now, we don't have a full outbound adapter wired, so this is left nil.
// When the outbound adapter is ready, wire it here.
```

```
File: MessageForge.Backend/internal/messenger/app.go:271-272

outboundWorker:  outboundWorker,  // nil
```

**OutboundWorker definition:**

```
File: MessageForge.Backend/internal/messenger/outbound_worker.go:27-32

type OutboundWorker struct {
    db       dbPool
    adapter  messengerAdapter
    delivery deliveryReporter
    logger   *slog.Logger
}
```

**Fix:**
1. Create or identify an adapter that implements `messengerAdapter` interface (likely `telegram.Adapter`)
2. Wire it into `NewApp()` when constructing the `OutboundWorker`
3. Add a startup warning log when outbound worker is disabled (nil adapter)

**Dependencies:** Requires FIX-001 (JSON contract) to be resolved first — no point sending if deserialization fails.

**Test:** Send a message from Salesforce LWC. Verify it appears in Telegram via the bot.

---

## Tier 1 — Before MVP Launch

These issues don't block basic testing but must be resolved before any customer-facing deployment.

---

### FIX-004: Inbound Queue Is a Dead Path — No Durability

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Status** | OPEN |
| **Component** | Go Backend |
| **Discovered** | 2026-03-30 (Architecture agent) |

**Problem:** The `inbound_queue` table exists in the schema. The `QueueWorker` is instantiated, registered as an `oklog/run` actor, and polls every second. But **nothing writes to `inbound_queue`**. Zero `INSERT INTO inbound_queue` statements exist in the entire Go codebase.

The real-time Pipeline handles all inbound messages via an in-memory channel (buffer size 256). When the buffer is full or `processMessage()` fails, the message is logged and dropped permanently.

**Evidence — no inserts found:**

```
Searched: MessageForge.Backend/**/*.go
Pattern:  INSERT INTO inbound_queue
Results:  0 matches
```

**Pipeline channel (in-memory only):**

```
File: MessageForge.Backend/internal/messenger/pipeline.go:41-59

func (p *Pipeline) Run(ctx context.Context) error {
    ch := p.source.Messages()
    for {
        select {
        case <-ctx.Done():
            return nil
        case msg, ok := <-ch:
            // ... process or drop
```

**Fix options:**

- **Option A (recommended):** Insert into `inbound_queue` in the webhook handler before returning 200 OK. Pipeline reads from queue instead of channel. Provides durability.
- **Option B:** Insert into `inbound_queue` on Pipeline failure (fallback). Keeps hot path fast but adds retry for failures.
- **Option C:** Document as "best-effort real-time" and remove the dead `QueueWorker` code to avoid confusion.

**Impact:** Under burst load or Salesforce downtime, inbound messages are permanently lost.

**Test:** Kill the Salesforce connection mid-message-flow. Verify messages are retried from the queue after recovery.

---

### FIX-005: Encrypted Credentials Not Wired

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Status** | OPEN (infrastructure exists, not integrated) |
| **Component** | Go Backend |
| **Discovered** | 2026-03-30 (Security agent) |

**Problem:** Migration 003 adds `encrypted_credentials BYTEA` column. `crypto.go` implements AES-256-GCM `Encrypt()`/`Decrypt()`. But no application code reads from or writes to `encrypted_credentials`. Bot tokens remain in plaintext JSONB in the `credentials` column.

**Encryption functions (exist but unused):**

```
File: MessageForge.Backend/internal/database/crypto.go:15-37 (Encrypt)
File: MessageForge.Backend/internal/database/crypto.go:41-73 (Decrypt)
```

**Migration (column added but not used):**

```
File: MessageForge.Backend/internal/database/migrations/003_encryption.sql
-- Adds encrypted_credentials BYTEA column
```

**No queries reference encrypted_credentials:**

```
Searched: MessageForge.Backend/**/*.go (excluding migrations/)
Pattern:  encrypted_credentials
Results:  0 matches
```

**Fix:**
1. Wire `Encrypt()` into the connection write path (INSERT/UPDATE `encrypted_credentials`)
2. Wire `Decrypt()` into the connection read path (SELECT + decrypt)
3. Add a data migration to encrypt existing plaintext credentials
4. Drop or nullify the plaintext `credentials` column after migration
5. Add `validate:"required"` to `EncryptionKey` config field (see FIX-006)

**Related:** FIX-006 (EncryptionKey validation)

**Test:** Insert a connection. Query the database directly. Verify `credentials` is null/empty and `encrypted_credentials` contains ciphertext.

---

### FIX-006: EncryptionKey Config Field Has No Validation

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Status** | OPEN |
| **Component** | Go Backend |
| **Discovered** | 2026-03-30 (Go reviewer agent) |

**Problem:** The `EncryptionKey` field in config has no `validate:"required"` tag. The server can start without an encryption key. When FIX-005 wires encryption into the credential lifecycle, a missing key will cause silent failures or panics.

**Current config:**

```
File: MessageForge.Backend/internal/messenger/config/config.go:48

EncryptionKey string `env:"ENCRYPTION_KEY"` // AES-256 key for encrypting bot tokens at rest
```

**Compare with WebhookSecret (correctly required):**

```
File: MessageForge.Backend/internal/messenger/config/config.go:45

WebhookSecret string `env:"WEBHOOK_SECRET" validate:"required"`
```

**Fix:** Add `validate:"required"` tag:

```go
EncryptionKey string `env:"ENCRYPTION_KEY" validate:"required"`
```

**Dependency:** Must be done before or alongside FIX-005.

---

### FIX-007: `archived/` Directory Breaks `go vet ./...`

| Field | Value |
|---|---|
| **Severity** | MEDIUM |
| **Status** | OPEN |
| **Component** | Go Backend |
| **Discovered** | 2026-03-30 (Build agent) |

**Problem:** The `archived/` directory sits inside the Go module root. It contains `.go` files with broken imports (`github.com/shaba/messenger-sf/internal/config`, `github.com/shaba/messenger-sf/internal/adapter`) that no longer exist. Running `go vet ./...` fails because the glob includes archived packages.

**Location:**

```
File: MessageForge.Backend/archived/backend/cmd/server/main.go:11
      — imports github.com/shaba/messenger-sf/internal/config (does not exist)

File: MessageForge.Backend/archived/backend/internal/salesforce/ingestion.go:11
      — imports github.com/shaba/messenger-sf/internal/adapter (does not exist)
```

**Fix options:**

- **Option A (recommended):** Move `archived/` outside the module root (e.g., to `MessageForge.Documentation/archived/`)
- **Option B:** Delete `archived/` entirely (it's in git history if needed)
- **Option C:** Add a `go.mod` file inside `archived/` to make it a separate module

**Test:** Run `go vet ./...` from `MessageForge.Backend/`. Should pass with zero output.

---

## Tier 2 — Before Production

These are code quality and robustness issues. Address before any production deployment or AppExchange submission.

---

### FIX-008: Terminal Error String Matching Is Fragile

| Field | Value |
|---|---|
| **Severity** | MEDIUM |
| **Status** | OPEN |
| **Component** | Go Backend |
| **Discovered** | 2026-03-30 (Go reviewer agent) |

**Problem:** Terminal error classification uses `strings.Contains(err.Error(), ...)` against a hardcoded slice. This is fragile — a wrapped error containing a sentinel string by coincidence will be misclassified. Additionally, the terminal error list is duplicated in two locations with different entries.

**Location 1 — outbound_worker.go (slice):**

```
File: MessageForge.Backend/internal/messenger/outbound_worker.go:213-228

func isTerminalSendError(err error) bool {
    msg := err.Error()
    terminals := []string{
        "USER_BANNED",
        "PEER_ID_INVALID",
        "CHAT_WRITE_FORBIDDEN",
        "BOT_BLOCKED",
        "USER_DEACTIVATED",
    }
```

**Location 2 — delivery_status.go (map):**

```
File: MessageForge.Backend/internal/salesforce/delivery_status.go:17-24

var terminalErrors = map[string]bool{
    "USER_BANNED":            true,
    "USER_BANNED_IN_CHANNEL": true,   // <-- missing from outbound_worker.go
    "PEER_ID_INVALID":        true,
    "CHAT_WRITE_FORBIDDEN":   true,
    "BOT_BLOCKED":            true,
    "USER_DEACTIVATED":       true,
}
```

**Discrepancy:** `delivery_status.go` includes `USER_BANNED_IN_CHANNEL`; `outbound_worker.go` does not.

**Fix:**
1. Define a `TelegramError` struct with a `Code` field in `telegram/errors.go`
2. Have the Telegram adapter return typed errors
3. Match with `errors.As` instead of string comparison
4. Single source of truth for terminal error codes

---

### FIX-009: GlobalMetrics Singleton — Layering Violation

| Field | Value |
|---|---|
| **Severity** | MEDIUM |
| **Status** | OPEN |
| **Component** | Go Backend |
| **Discovered** | 2026-03-30 (Go reviewer agent) |

**Problem:** `server.GlobalMetrics` is a mutable package-level singleton. Workers in the `messenger` package import the `server` package to access it — a layering violation (`messenger` -> `server`). Tests cannot isolate metrics.

**Singleton declaration:**

```
File: MessageForge.Backend/internal/server/metrics.go:26

var GlobalMetrics = &Metrics{}
```

**Cross-package imports:**

```
File: MessageForge.Backend/internal/messenger/outbound_worker.go:11 — imports "server"
     Lines 146, 158 — server.GlobalMetrics.OutboundError.Add(1)

File: MessageForge.Backend/internal/messenger/queue_worker.go:11 — imports "server"
     Lines 129, 136 — server.GlobalMetrics.InboundError.Add(1)
```

**Fix:** Pass `*Metrics` as a constructor parameter to both workers. Remove the global variable.

---

### FIX-010: ContentType Not Allowlisted in Outbound Media

| Field | Value |
|---|---|
| **Severity** | MEDIUM |
| **Status** | OPEN (handler not yet registered) |
| **Component** | Go Backend |
| **Discovered** | 2026-03-30 (Go reviewer agent) |

**Problem:** `req.ContentType` is passed directly to R2's presign API without validation. An attacker with a valid HMAC token could set `Content-Type: text/html`, enabling stored XSS via the CDN.

**Location:**

```
File: MessageForge.Backend/internal/media/outbound.go:49-52

if req.ChatID == "" || req.ContentType == "" || req.Filename == "" {
    http.Error(w, "chat_id, content_type, and filename are required", http.StatusBadRequest)
    return
}
```

```
File: MessageForge.Backend/internal/media/outbound.go:64

url, err := h.presigner.PresignedPutURL(r.Context(), key, req.ContentType, h.uploadTTL)
```

**Fix:** Add an allowlist of safe content types before passing to presigner:

```go
var allowedContentTypes = map[string]bool{
    "image/jpeg": true, "image/png": true, "image/gif": true, "image/webp": true,
    "video/mp4": true, "audio/ogg": true, "audio/mpeg": true,
    "application/pdf": true, "application/zip": true,
}
```

**Timing:** Must be done before this handler is registered in `app.go`.

---

### FIX-011: Chat_SF_ID__c Field Naming Ambiguity

| Field | Value |
|---|---|
| **Severity** | MEDIUM |
| **Status** | OPEN |
| **Component** | Salesforce |
| **Discovered** | 2026-03-30 (Salesforce safety agent) |

**Problem:** The `Chat_SF_ID__c` field on `Inbound_Message__e` is labeled "Chat SF ID" with length 18 (Salesforce ID size), but `InboundMessageTriggerHandler` uses it for both Salesforce IDs and external IDs. If an external ID happens to be 18 alphanumeric characters, it could parse as a valid (but wrong) Salesforce ID.

**Dual-purpose usage:**

```
File: MessageForge.Salesforce/force-app/main/default/classes/InboundMessageTriggerHandler.cls:25-32

try {
    sfChatIds.add(Id.valueOf(evt.Chat_SF_ID__c));   // Treated as SF ID
} catch (StringException ex) {
    externalChatIds.add(evt.Chat_SF_ID__c);          // Treated as external ID
}
```

**Fix options:**

- **Option A:** Split into two fields: `Chat_SF_ID__c` (18-char SF ID) and `Chat_External_ID__c` (variable-length external ID). Go sends to the appropriate field.
- **Option B:** Rename to `Chat_Reference__c`, increase max length, and document that it accepts both formats.
- **Option C:** Keep current behavior, document the dual-purpose nature, and add a prefix convention (e.g., `ext:` prefix for external IDs).

---

### FIX-012: `console.error()` Statements in Production LWC

| Field | Value |
|---|---|
| **Severity** | MEDIUM |
| **Status** | OPEN |
| **Component** | Salesforce |
| **Discovered** | 2026-03-30 (Salesforce safety agent) |

**Problem:** 7 `console.error()` calls remain in production LWC code. These expose debugging information (error objects, stack traces) to anyone with browser DevTools open.

**Locations:**

```
File: MessageForge.Salesforce/force-app/main/default/lwc/messengerChat/messengerChat.js
  Line 51:  console.error('empApi error:', error);
  Line 73:  console.error('Inbound subscribe failed:', error);
  Line 92:  console.error('Delivery status subscribe failed:', error);
  Line 198: console.error('Error loading messages:', error);
  Line 282: console.error('Send failed:', error);

File: MessageForge.Salesforce/force-app/main/default/lwc/messengerLiveChat/messengerLiveChat.js
  Line 28:  console.error('Failed to load Centrifugo URL:', err);
  Line 56:  console.error('Centrifugo:', context, err);
```

**Fix:** Remove or replace with a production-safe logging utility. AppExchange security review may flag these.

---

## Tier 3 — Code Quality / Low Priority

---

### FIX-013: Transaction Rollback Uses Cancelled Context

| Field | Value |
|---|---|
| **Severity** | LOW |
| **Status** | OPEN |
| **Component** | Go Backend |

**Problem:** On SIGINT, the deferred `Rollback(ctx)` runs with an already-cancelled context. pgx returns `context.Canceled` and the error is discarded. PostgreSQL rolls back server-side when the connection closes, so no data corruption occurs — but the code is misleading.

**Locations:**

```
File: MessageForge.Backend/internal/messenger/queue_worker.go:83

defer func() {
    if !committed {
        _ = dbTx.Rollback(ctx)   // ctx is already cancelled
    }
}()

File: MessageForge.Backend/internal/messenger/outbound_worker.go:65
// Same pattern
```

**Fix:** Use `context.Background()` with a short timeout for the rollback call.

---

### FIX-014: DML Exception Logging Lacks Context

| Field | Value |
|---|---|
| **Severity** | LOW |
| **Status** | OPEN |
| **Component** | Salesforce |

**Problem:** Trigger handlers catch `DmlException` but log only the generic message. No field names, index, or status codes. Makes production troubleshooting difficult.

**Locations:**

```
File: MessageForge.Salesforce/force-app/main/default/classes/InboundMessageTriggerHandler.cls:130
File: MessageForge.Salesforce/force-app/main/default/classes/DeliveryStatusTriggerHandler.cls:89
File: MessageForge.Salesforce/force-app/main/default/classes/SessionStatusTriggerHandler.cls:64
```

**Fix:** Include `getDmlFieldNames()`, `getDmlStatusCode()`, and record index in the debug output.

---

### FIX-015: Missing Bulk Test Scenarios

| Field | Value |
|---|---|
| **Severity** | LOW |
| **Status** | OPEN |
| **Component** | Salesforce |

**Problem:** No explicit bulk operation tests (200+ records) visible in the test classes. Salesforce governor limits require bulk-safe code, which should be verified by tests.

**Fix:** Add test methods that process 200+ records for each trigger handler.

---

## Test Coverage Gaps (Reference)

Current Go test coverage is 50.1% against 80% target.

| Package | Coverage | Critical Gaps |
|---|---|---|
| `internal/messenger` | 16.8% | `app.go` (NewApp, Run), `queue_worker.go`, `outbound_worker.go` — all 0% |
| `internal/database` | 38.4% | `migrations.go`, `session_backup.go`, `PgSessionStore` — all 0% |
| `internal/server` | 49.0% | `server.go` (RegisterWebhook, AdminRun), `metrics.go` — all 0% |
| `internal/media` | 62.9% | `r2.go` (Upload, Download, Delete, Presign) — all 0% |
| `internal/telegram/botapi` | 76.8% | Near target |
| `internal/telegram/mtproto` | 77.4% | Near target |
| `internal/salesforce` | 79.9% | Marginal — just under 80% |
| `internal/realtime` | 86.4% | OK |
| `internal/telegram` | 87.7% | OK |
| `internal/messenger/config` | 100% | OK |

**Priority:** `internal/messenger` and `internal/database` are the most critical gaps — they contain the core application wiring and data persistence.

---

## Cross-Reference to Prior Reviews

| Fix | Original Review | Original ID |
|---|---|---|
| FIX-001 | Architecture agent (2026-03-30) | New finding |
| FIX-002 | Salesforce safety agent (2026-03-30) | New finding |
| FIX-003 | Architecture agent (2026-03-30) | New finding |
| FIX-004 | [Recheck Review](./2026-03-30-recheck-review.md) | NM3 (upgraded to HIGH) |
| FIX-005 | [Recheck Review](./2026-03-30-recheck-review.md) | NH6 |
| FIX-006 | Go reviewer agent (2026-03-30) | S5 |
| FIX-007 | Build agent (2026-03-30) | New finding |
| FIX-008 | [Recheck Review](./2026-03-30-recheck-review.md) | LOW (terminal errors) |
| FIX-009 | [Recheck Review](./2026-03-30-recheck-review.md) | LOW (GlobalMetrics) |
| FIX-010 | Go reviewer agent (2026-03-30) | Q3 |
| FIX-011 | Salesforce safety agent (2026-03-30) | New finding |
| FIX-012 | Salesforce safety agent (2026-03-30) | New finding |
| FIX-013 | Go reviewer agent (2026-03-30) | C2 |
| FIX-014 | Salesforce safety agent (2026-03-30) | New finding |
| FIX-015 | Salesforce safety agent (2026-03-30) | New finding |

---

## Summary

| Tier | Count | Severity | Action |
|---|---|---|---|
| **Tier 0** | 3 | CRITICAL | Fix immediately — blocks all integration testing |
| **Tier 1** | 4 | HIGH | Fix before MVP launch |
| **Tier 2** | 5 | MEDIUM | Fix before production / AppExchange submission |
| **Tier 3** | 3 | LOW | Fix when convenient |
| **Total** | **15** | | |

**Estimated total effort:** 20-30 hours

| Tier | Estimated Effort |
|---|---|
| Tier 0 | 4-6 hours |
| Tier 1 | 8-12 hours |
| Tier 2 | 6-8 hours |
| Tier 3 | 2-4 hours |
