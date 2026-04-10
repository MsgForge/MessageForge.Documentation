# Phase 2: Telegram Bot via Go Gateway

> Implementation spec for a workable Telegram Bot inbound flow through Go Gateway.
> All other channels (WhatsApp, Messenger, Instagram, Viber, Twilio) are tech debt.

## Architecture

```
Telegram Bot API
      |
      | POST /webhook/telegram_bot/{channelId}
      v
Go Gateway (messageforge-backend.fly.dev:8080)
      |
      |-- 1. Validate X-Telegram-Bot-Api-Secret-Token header
      |-- 2. Parse Telegram JSON -> NormalizedInboundDTO (Go struct)
      |-- 3. Persist to Postgres (status: pending)
      |-- 4. Respond 200 to Telegram immediately
      |-- 5. Enqueue to bounded worker pool (async processing)
      |       a. If attachments: check fileSize < 10 MB before downloading
      |       b. Download from Telegram -> Upload to SF ContentVersion (multipart) -> Get SF IDs
      |       c. POST normalized DTO to SF Apex REST
      |-- 6. Update Postgres (status: delivered)
      v
Salesforce (Apex REST)
      |
      |-- POST /services/apexrest/tgint/gateway-inbound
      |-- Deduplicate by Message_External_ID__c per chat
      |-- Upsert Chat, Insert Message, Insert Attachments (with Content_Version_ID__c)
      |-- Publish Inbound_Message__e Platform Event (chunked at 150 per batch)
      v
LWC Chat UI (real-time via empApi)
      |
      |-- Displays text, photo, video, voice, document, sticker
      |-- Media viewer modal with download
```

Outbound (SF -> Telegram) continues to work natively via `TelegramSender.cls` calling Telegram API directly. No Go Gateway involvement for outbound.

**Design decision — asymmetric architecture:** Inbound flows through Go Gateway (required for AppExchange — SF Sites cannot be packaged). Outbound stays native in SF (no latency benefit to routing through Go). The bot token is stored in both SF (`Telegram_Bot_Config__c.Bot_Token__c`) and Go (`channel_configs.config_encrypted`). When the token is rotated, both must be updated via the channel config sync mechanism (see section 3.2).

---

## Work Stream 1: Go Gateway

### 1.1 Project Structure

```
MessageForge.Gateway/
  cmd/
    messenger/
      main.go                    # Entry point
  internal/
    config/
      config.go                  # Env var loading
    crypto/
      webhook_auth.go            # Telegram secret token validation (constant-time)
      webhook_auth_test.go
      encryption.go              # AES-256-GCM for Postgres encryption
      encryption_test.go
    models/
      inbound.go                 # NormalizedInboundDTO, NormalizedAttachmentDTO
    providers/
      telegram/
        parser.go                # Telegram JSON -> DTO
        parser_test.go
        downloader.go            # Telegram getFile + download
        downloader_test.go
    persistence/
      migrations/
        001_initial_schema.sql   # Initial Postgres DDL
      db.go                      # Connection pool, migration runner
      message_repo.go            # CRUD for messages table
      message_repo_test.go
      config_repo.go             # CRUD for channel_configs table
      config_repo_test.go
    salesforce/
      client.go                  # JWT Bearer OAuth2 client
      client_test.go
      content_upload.go          # Upload files as ContentVersion (multipart)
      content_upload_test.go
      inbound_sender.go          # POST DTO to SF endpoint
      inbound_sender_test.go
    handlers/
      webhook.go                 # POST /webhook/{channelType}/{channelId}
      webhook_test.go
      health.go                  # GET /health (deep check: DB + SF)
      config.go                  # POST /api/channel-config (receive config from SF)
      config_test.go
    worker/
      pool.go                    # Bounded worker pool (channel-based)
      pool_test.go
    retry/
      worker.go                  # Retry failed/pending messages
      worker_test.go
      backoff.go                 # Exponential backoff with jitter
      backoff_test.go
  go.mod
  go.sum
  Dockerfile
  docker-compose.yml
  fly.toml
  .env.example
```

### 1.1.1 Dependencies

The following Go dependencies are required. Choose during implementation:

| Dependency | Purpose | Notes |
|-----------|---------|-------|
| `github.com/jackc/pgx/v5` | Postgres driver | Preferred over `lib/pq`: native connection pooling via `pgxpool`, better performance, batch support |
| `golang.org/x/crypto` | `subtle.ConstantTimeCompare` | For webhook secret validation |
| `golang.org/x/sync/errgroup` | Goroutine lifecycle | Used for worker pool and graceful shutdown |
| `golang.org/x/time/rate` | Rate limiting | Per-IP rate limiter on webhook endpoint |
| `github.com/golang-migrate/migrate/v4` | Schema migrations | Idempotent, versioned migrations from day one |
| `log/slog` (stdlib) | Structured logging | JSON handler for Fly.io log aggregation (Go 1.21+, included in Go 1.23) |

**Do NOT add**: external routers (stdlib `http.ServeMux` with Go 1.22+ path params is sufficient), ORMs (raw SQL with `pgx` is fine for this scope).

### 1.2 Config (`internal/config/config.go`)

Load from environment variables. Panic on missing required values at startup.

```go
type Config struct {
    // Server
    Port string // default "8080"

    // Salesforce OAuth2 (JWT Bearer Flow)
    SalesforceLoginURL    string // e.g. "https://login.salesforce.com" (or "https://test.salesforce.com" for sandbox)
    SalesforceClientID    string // Connected App consumer key
    SalesforceUsername    string // API user username
    SalesforcePrivateKey  string // PEM-encoded RSA private key for JWT signing (from env var or file path)

    // Postgres
    DatabaseURL string // e.g. "postgres://messenger:localdev@localhost:5432/messenger?sslmode=disable"

    // Encryption
    EncryptionKeys []EncryptionKeyEntry // Supports key rotation; newest key used for encrypt, all tried for decrypt

    // Gateway secret (shared with SF)
    GatewaySecret string // 256-bit random, hex-encoded. Used for X-Gateway-Secret header validation

    // Worker pool
    MaxWorkers int // default 20, max concurrent background processors

    // Log level
    LogLevel string // "debug", "info", "warn", "error"
}

// EncryptionKeyEntry supports key rotation.
// Env var format: ENCRYPTION_KEYS=v2:base64key2,v1:base64key1
// First entry is the active key (used for encryption). All entries tried for decryption.
type EncryptionKeyEntry struct {
    Version string // e.g. "v1", "v2"
    Key     []byte // 32 bytes (256-bit)
}
```

**JWT Bearer flow replaces Username-Password flow.** Rationale:
- Username-Password flow is deprecated by Salesforce and blocked by MFA enforcement (mandatory since Feb 2022)
- AppExchange security review rejects packages requiring stored passwords
- JWT Bearer uses an X.509 certificate (deployed with the Connected App) and a private key (stored in Fly.io secrets)
- No user password or security token ever leaves the org

### 1.3 Webhook Secret Validation (`internal/crypto/webhook_auth.go`)

> File renamed from `hmac.go` to `webhook_auth.go` — this is NOT HMAC validation. Telegram uses simple header equality.

```go
// ValidateWebhookSecret checks X-Telegram-Bot-Api-Secret-Token header
// against the stored webhook secret for this channel.
//
// The secret is loaded from Postgres channel_configs table (encrypted).
// Uses crypto/subtle.ConstantTimeCompare to prevent timing attacks.
func ValidateWebhookSecret(headerValue, storedSecret string) error
```

The stored secret comes from Postgres `channel_configs` table, decrypted at comparison time.

**Implementation requirement:** Use `subtle.ConstantTimeCompare([]byte(headerValue), []byte(storedSecret))`. Do NOT use `==` or `strings.EqualFold`.

### 1.4 Telegram Parser (`internal/providers/telegram/parser.go`)

Port of `TelegramInboundParser.cls`. Must produce identical normalized output.

**Input:** Raw Telegram webhook JSON (from `request.Body`)

**Telegram webhook JSON structure:**
```json
{
  "update_id": 123456789,
  "message": {
    "message_id": 42,
    "date": 1700000000,
    "chat": {
      "id": -1001234567890,
      "type": "supergroup",
      "title": "My Group"
    },
    "from": {
      "id": 987654321,
      "first_name": "John",
      "last_name": "Doe"
    },
    "text": "Hello world",
    "reply_to_message": { "message_id": 41 },
    "photo": [
      {"file_id": "small_id", "width": 90, "height": 90, "file_size": 1234},
      {"file_id": "large_id", "width": 800, "height": 600, "file_size": 45678}
    ],
    "caption": "Photo caption",
    "document": {"file_id": "doc_id", "file_name": "report.pdf", "mime_type": "application/pdf", "file_size": 102400},
    "video": {"file_id": "vid_id", "mime_type": "video/mp4", "width": 1920, "height": 1080, "duration": 30, "file_size": 5000000},
    "voice": {"file_id": "voice_id", "mime_type": "audio/ogg", "duration": 5, "file_size": 12000},
    "audio": {"file_id": "audio_id", "mime_type": "audio/mpeg", "duration": 180, "file_size": 3000000},
    "sticker": {"file_id": "sticker_id", "width": 512, "height": 512}
  }
}
```

Also handle `edited_message` (same structure, sets `IsEdited = true`).

**Output Go struct** (`internal/models/inbound.go`):

```go
type NormalizedInboundDTO struct {
    ChatExternalID    string                    `json:"chatExternalId"`
    ChatType          string                    `json:"chatType"`          // private|group|supergroup|channel
    ChatTitle         string                    `json:"chatTitle"`
    MessageExternalID string                    `json:"messageExternalId"`
    SenderExternalID  string                    `json:"senderExternalId"`
    SenderName        string                    `json:"senderName"`
    MessageText       string                    `json:"messageText"`
    MessageType       string                    `json:"messageType"`       // text|photo|video|voice|document|sticker
    SentAt            string                    `json:"sentAt"`            // ISO 8601 datetime
    ReplyToExternalID string                    `json:"replyToExternalId"`
    IsEdited          bool                      `json:"isEdited"`
    Attachments       []NormalizedAttachmentDTO  `json:"attachments"`
}

type NormalizedAttachmentDTO struct {
    MediaExternalID    string `json:"mediaExternalId"`    // Telegram file_id
    DownloadURL        string `json:"downloadUrl"`        // Resolved download URL (empty initially)
    MimeType           string `json:"mimeType"`
    FileName           string `json:"fileName"`
    FileSize           int64  `json:"fileSize"`
    SortOrder          int    `json:"sortOrder"`
    DurationSeconds    int    `json:"durationSeconds"`
    Width              int    `json:"width"`
    Height             int    `json:"height"`
    ContentVersionID   string `json:"contentVersionId"`   // SF ID after upload
}
```

**Parsing rules (MUST match SF parser exactly):**

| Rule | Detail |
|------|--------|
| Chat ID | Use int64, convert to string (can exceed int32 max) |
| Photo selection | Always use LAST element in `photo` array (highest resolution) |
| Chat title | Use `chat.title` for groups; fall back to `chat.first_name` for private |
| Sender name | `first_name` + space + `last_name` (if present) |
| Caption vs text | If media message has `caption`, use caption as `messageText`; else use `text` |
| Date conversion | Unix epoch seconds -> ISO 8601 (e.g. `2024-01-15T10:30:00.000Z`) |
| Message type | Detect media type in priority: photo > document > video > voice > audio > sticker > text |
| Audio normalization | Telegram `audio` type maps to `voice` in our DTO |
| Sticker MIME | Always `image/webp` |
| Reply threading | Extract `reply_to_message.message_id` as string |
| Edited messages | Parse `edited_message` same as `message`, set `isEdited = true` |

**Edited message SF behavior:** When SF receives a DTO with `isEdited = true`, the existing `InboundDataService.processInboundMessages()` inserts a new `Messenger_Message__c` record (it does NOT update the original). The `isEdited` flag is stored on the message record for UI display. This is intentional — chat history is append-only. The LWC should display an "(edited)" label on edited messages. Verify this behavior in E2E tests.

### 1.5 Telegram File Downloader (`internal/providers/telegram/downloader.go`)

Two-step process matching `MediaDownloadService.cls`:

```
Step 1: GET https://api.telegram.org/bot{token}/getFile?file_id={file_id}
        Response: {"ok": true, "result": {"file_path": "photos/file_42.jpg", "file_size": 45678}}

Step 2: GET https://api.telegram.org/file/bot{token}/{file_path}
        Response: Raw file bytes
```

**Size limit — CRITICAL: check BEFORE downloading:**
- **10 MB hard limit** (not 20 MB). Rationale: SF REST API ContentVersion upload via multipart/form-data has a ~12 MB effective limit, but we use 10 MB as a safe margin.
- Check `file_size` from the Telegram webhook payload (available in `NormalizedAttachmentDTO.FileSize`) **before** calling `getFile`/download. If `fileSize > 10*1024*1024`, skip the attachment immediately — do NOT download.
- For photos, Telegram's `file_size` in the webhook payload may be approximate. Add a secondary check: after download, if actual bytes exceed 10 MB, discard and mark attachment as `skipped`.
- Log a warning: `"attachment skipped: file too large"` with `file_size`, `file_id`, `channel_id`.
- The text/caption portion of the message is still delivered to SF. Only the attachment is skipped.

**Bot token:** Retrieved from Postgres `channel_configs.config_encrypted` (decrypt before use).

```go
type TelegramDownloader struct {
    httpClient *http.Client // shared client, 30s timeout, no redirect following
}

// ResolveAndDownload resolves file_id to URL, downloads bytes.
// Returns file bytes, resolved URL, and error.
// Rejects files > maxSize bytes. Uses io.LimitReader to prevent OOM.
func (d *TelegramDownloader) ResolveAndDownload(ctx context.Context, fileID, botToken string, maxSize int64) ([]byte, string, error)
```

**Implementation requirements:**
- Wrap download response body with `io.LimitReader(resp.Body, maxSize+1)` — if read returns `maxSize+1` bytes, the file exceeds the limit. Discard and return error.
- Set `httpClient.CheckRedirect = func(...) error { return http.ErrUseLastResponse }` — do not follow redirects.
- Reuse the same `*http.Client` for both `getFile` API call and file download (connection pooling).
- Use the `ctx` parameter for all HTTP calls — do NOT use a cancelled request context (see section 1.11).

### 1.6 Postgres Schema (`internal/persistence/migrations/001_initial_schema.sql`)

```sql
CREATE TABLE IF NOT EXISTS inbound_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id      TEXT NOT NULL,               -- SF Messenger_Channel__c ID
    channel_type    TEXT NOT NULL DEFAULT 'Telegram_Bot',
    raw_payload     BYTEA,                       -- Encrypted original webhook JSON
    normalized_dto  BYTEA,                       -- Encrypted normalized DTO (contains PII)
    status          TEXT NOT NULL DEFAULT 'pending',  -- pending|processing|delivered|failed|dead
    attempt_count   INT NOT NULL DEFAULT 0,
    retry_after     TIMESTAMPTZ,                 -- NULL = immediately eligible; set by MarkFailed
    last_error      TEXT,
    sf_message_id   TEXT,                        -- SF Messenger_Message__c ID (after delivery)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inbound_messages_status ON inbound_messages(status, retry_after)
    WHERE status IN ('pending', 'failed');
CREATE INDEX idx_inbound_messages_channel ON inbound_messages(channel_id);

CREATE TABLE IF NOT EXISTS inbound_attachments (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id            UUID NOT NULL REFERENCES inbound_messages(id) ON DELETE CASCADE,
    media_external_id     TEXT NOT NULL,          -- Telegram file_id
    mime_type             TEXT,
    file_name             TEXT,
    file_size             BIGINT,
    sort_order            INT NOT NULL DEFAULT 1,
    file_body             BYTEA,                  -- Downloaded file bytes (encrypted, cleared after SF upload)
    sf_content_version_id TEXT,                   -- SF ContentVersion ID (after upload)
    status                TEXT NOT NULL DEFAULT 'pending', -- pending|downloaded|uploaded|skipped|failed
    last_error            TEXT,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inbound_attachments_message ON inbound_attachments(message_id);
CREATE INDEX idx_inbound_attachments_status ON inbound_attachments(status) WHERE status IN ('pending', 'downloaded');

CREATE TABLE IF NOT EXISTS channel_configs (
    id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id             TEXT NOT NULL UNIQUE,   -- SF Messenger_Channel__c ID
    channel_type           TEXT NOT NULL,           -- Telegram_Bot (extensible for future channels)
    config_encrypted       BYTEA NOT NULL,          -- AES-256-GCM encrypted JSON blob (channel-specific)
    is_active              BOOLEAN NOT NULL DEFAULT true,
    updated_at             TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channel_configs_channel ON channel_configs(channel_id);
```

**Encryption strategy:**

| Field | Type | Encrypted? | Rationale |
|-------|------|-----------|-----------|
| `raw_payload` | BYTEA | Yes (AES-256-GCM) | Contains raw Telegram JSON with user data |
| `normalized_dto` | BYTEA | Yes (AES-256-GCM) | Contains PII: `senderExternalId`, `senderName`, `chatExternalId` (GDPR) |
| `file_body` | BYTEA | Yes (AES-256-GCM) | User-uploaded file content |
| `config_encrypted` | BYTEA | Yes (AES-256-GCM) | Contains bot tokens, webhook secrets — highest sensitivity |

**AES-256-GCM nonce/IV strategy (CRITICAL — must follow exactly):**

1. **Nonce generation:** Generate a cryptographically random 12-byte nonce via `crypto/rand.Read` for EVERY encrypt call. Never reuse a nonce with the same key — GCM is catastrophically broken on nonce reuse (key stream recovery).
2. **Ciphertext format:** `key_version_byte || 12-byte-nonce || ciphertext || 16-byte-GCM-tag`. The key version byte allows decryption with the correct key during rotation.
3. **Decryption:** Read the first byte to identify the key version, extract the 12-byte nonce, decrypt with the corresponding key.
4. **Key rotation:** When `ENCRYPTION_KEYS` env var is updated with a new key version, new data is encrypted with the newest key. Existing data remains decryptable because old keys are kept in the list. To fully rotate off an old key, run a background migration that re-encrypts all rows with the new key, then remove the old key from the env var.
5. **Key format in env var:** `ENCRYPTION_KEYS=v2:base64encodedkey,v1:base64encodedkey`. First entry = active encryption key. All entries used for decryption attempts.
6. **file_body lifecycle:** Cleared (set to NULL) after successful SF ContentVersion upload to save space.

**Schema migration strategy:** Use `golang-migrate/migrate` with embedded SQL files. Even for this initial single migration, the tooling investment avoids accumulating migration debt. Future schema changes (adding columns, etc.) are handled via numbered migration files (`002_add_column.sql`) with both up and down migrations.

**channel_configs.config_encrypted — extensible JSON blob:**

Instead of channel-type-specific columns (`bot_token_encrypted`, `webhook_secret_encrypted`), store a single encrypted JSON blob. This avoids schema changes when adding new channel types.

```go
// Telegram config (decrypted from config_encrypted)
type TelegramChannelConfig struct {
    BotToken      string `json:"botToken"`
    WebhookSecret string `json:"webhookSecret"`
}

// Future: WhatsApp config would be a different struct
// type WhatsAppChannelConfig struct { ... }
```

**Postgres connection pool configuration (REQUIRED):**

```go
poolConfig, _ := pgxpool.ParseConfig(cfg.DatabaseURL)
poolConfig.MaxConns = 25           // Fly.io managed PG limits connections
poolConfig.MinConns = 5
poolConfig.MaxConnLifetime = 5 * time.Minute
poolConfig.MaxConnIdleTime = 30 * time.Second
```

### 1.7 Message Repository (`internal/persistence/message_repo.go`)

```go
type MessageRepository struct {
    pool      *pgxpool.Pool
    encrypter *crypto.Encrypter
}

// SaveInboundWithAttachments persists a new inbound message AND its attachments
// in a SINGLE database transaction. Returns the generated message UUID.
// CRITICAL: Must use pgx.Tx — if attachment insert fails after message insert,
// both are rolled back. Prevents orphaned message rows without attachments.
func (r *MessageRepository) SaveInboundWithAttachments(
    ctx context.Context,
    channelID string,
    rawPayload []byte,
    dto *models.NormalizedInboundDTO,
) (string, error)

// SetProcessing atomically sets status to "processing" using SELECT FOR UPDATE SKIP LOCKED.
// Returns the message row if successfully claimed, nil if already claimed by another worker.
// CRITICAL: Prevents retry worker and webhook goroutine from processing the same message.
func (r *MessageRepository) SetProcessing(ctx context.Context, messageID string) (*InboundMessageRow, error)

// UpdateAttachmentFile stores downloaded file body (encrypted) and updates status to "downloaded".
func (r *MessageRepository) UpdateAttachmentFile(ctx context.Context, attID string, fileBody []byte) error

// UpdateAttachmentSFID sets sf_content_version_id, clears file_body, updates status to "uploaded".
func (r *MessageRepository) UpdateAttachmentSFID(ctx context.Context, attID string, sfContentVersionID string) error

// SkipAttachment sets status to "skipped" with a reason (e.g., "file too large").
func (r *MessageRepository) SkipAttachment(ctx context.Context, attID string, reason string) error

// MarkDelivered sets message status to "delivered" and stores SF message ID.
func (r *MessageRepository) MarkDelivered(ctx context.Context, messageID string, sfMessageID string) error

// MarkFailed increments attempt_count, sets last_error, sets retry_after.
// Backoff schedule: 30s, 2min, 5min, 15min, 30min, 1h, 2h, 4h, 8h, 12h.
// After 10 attempts, status -> "dead".
func (r *MessageRepository) MarkFailed(ctx context.Context, messageID string, err string) error

// GetRetryable returns messages eligible for retry:
// status = 'failed' AND retry_after <= now() AND attempt_count < 10.
// Uses SELECT FOR UPDATE SKIP LOCKED to prevent concurrent pickup.
// Does NOT return status='processing' (those are in-flight).
func (r *MessageRepository) GetRetryable(ctx context.Context, limit int) ([]InboundMessageRow, error)
```

**Backoff schedule** (defined as package-level variable in `internal/retry/backoff.go`):

> Note: Go does not support `const` for slices, so this is a `var`. Treat it as immutable — never reassign at runtime.

```go
var BackoffSchedule = []time.Duration{
    30 * time.Second,  // attempt 1
    2 * time.Minute,   // attempt 2
    5 * time.Minute,   // attempt 3
    15 * time.Minute,  // attempt 4
    30 * time.Minute,  // attempt 5
    1 * time.Hour,     // attempt 6
    2 * time.Hour,     // attempt 7
    4 * time.Hour,     // attempt 8
    8 * time.Hour,     // attempt 9
    12 * time.Hour,    // attempt 10 -> dead
}
```

**Rationale for 10 attempts / ~24h total:** Salesforce maintenance windows typically last 15-60 minutes. The old 5-attempt / 13-minute schedule would mark messages as "dead" during a routine maintenance window. 10 attempts spanning ~24 hours ensures messages survive extended outages.

### 1.8 Salesforce JWT Bearer OAuth2 Client (`internal/salesforce/client.go`)

**Use JWT Bearer flow** — NOT Username-Password flow (which is deprecated and blocked by MFA enforcement).

**JWT Bearer flow:**

```
1. Build JWT assertion:
   - Header: {"alg": "RS256"}
   - Payload:
     - iss: Connected App consumer key (client_id)
     - sub: API user username
     - aud: "https://login.salesforce.com" (or "https://test.salesforce.com")
     - exp: current time + 300 seconds
   - Sign with RSA private key (PKCS#8 PEM format)

2. POST https://{login_url}/services/oauth2/token
   Content-Type: application/x-www-form-urlencoded

   grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
   &assertion={signed_jwt}

3. Response: {"access_token": "...", "instance_url": "https://myorg.my.salesforce.com", ...}
```

```go
type SalesforceClient struct {
    instanceURL string
    accessToken string
    httpClient  *http.Client  // 30s timeout, shared across all calls
    cfg         *config.Config
    mu          sync.Mutex    // protects token refresh; Mutex (not RWMutex) because the
                              // critical section is always a write (refresh). RWMutex adds
                              // no benefit — there is no read-only path through the lock.
    tokenExpiry time.Time     // track when token was last refreshed
}

// NewSalesforceClient authenticates via JWT Bearer and returns a ready client.
func NewSalesforceClient(cfg *config.Config) (*SalesforceClient, error)

// refreshToken re-authenticates via JWT Bearer when 401 received.
// Uses compare-and-refresh pattern to prevent thundering herd:
// 1. Acquire mutex
// 2. Check if token was already refreshed by another goroutine (compare tokenExpiry)
// 3. If already refreshed, return immediately (no duplicate OAuth call)
// 4. If not, perform new JWT assertion and token exchange
// 5. Update accessToken and tokenExpiry
func (c *SalesforceClient) refreshToken(ctx context.Context) error

// doRequest executes an HTTP request with auth header, auto-refreshes on 401.
// CRITICAL: ctx must be a background context with timeout (not a request context).
func (c *SalesforceClient) doRequest(ctx context.Context, method, path string, body io.Reader, contentType string) (*http.Response, error)
```

**Compare-and-refresh pattern (prevents thundering herd):**
When multiple goroutines receive 401 simultaneously, only the first one performs the OAuth call. Others wait on the mutex and see that `tokenExpiry` was updated, so they skip the refresh and retry with the new token. This prevents N goroutines from all issuing independent OAuth requests.

**SF API rate limit awareness:**
Log a warning when approaching API limits. Check `Sforce-Limit-Info` response header from SF:
```
Sforce-Limit-Info: api-usage=450/100000
```
If usage exceeds 80% of the limit, log at WARN level. If > 95%, log at ERROR and consider backoff.

### 1.9 ContentVersion Upload (`internal/salesforce/content_upload.go`)

Upload file to Salesforce as a ContentVersion record using **multipart/form-data** (NOT JSON with base64 `VersionData`).

**Rationale:** JSON with base64 inflates file size by ~33%, reducing the effective upload limit from 12 MB to ~9 MB. Multipart/form-data sends raw bytes and supports the full 12 MB.

```
POST https://{instance_url}/services/data/v62.0/sobjects/ContentVersion
Content-Type: multipart/form-data; boundary=boundary_string

--boundary_string
Content-Disposition: form-data; name="entity_content"
Content-Type: application/json

{"Title": "photo_1.jpg", "PathOnClient": "photo_1.jpg", "ContentLocation": "S"}
--boundary_string
Content-Disposition: form-data; name="VersionData"; filename="photo_1.jpg"
Content-Type: image/jpeg

{raw file bytes}
--boundary_string--
```

Response: `{"id": "068...", "success": true}`

Then query to get ContentDocumentId:
```
GET https://{instance_url}/services/data/v62.0/sobjects/ContentVersion/{id}?fields=ContentDocumentId
```

**Size limit:** 10 MB hard limit (checked before download — see section 1.5). Files that somehow exceed this after download are rejected before upload.

```go
type ContentUploadResult struct {
    ContentVersionID  string
    ContentDocumentID string
}

// UploadFile uploads bytes as a Salesforce ContentVersion using multipart/form-data.
// Returns ContentVersion ID and ContentDocument ID.
// CRITICAL: Do NOT use JSON with base64 VersionData — use multipart to preserve full 12 MB limit.
func (c *SalesforceClient) UploadFile(ctx context.Context, fileName string, mimeType string, body []byte) (*ContentUploadResult, error)
```

The ContentVersion download URL pattern for LWC display:
```
/sfc/servlet.shepherd/version/download/{ContentVersionID}
```

This URL is what gets set on the attachment's `contentVersionId` field in the DTO sent to SF. SF then constructs the full URL when setting `Media_URL__c`.

**ContentVersion idempotency note:** If Go uploads a ContentVersion and crashes before recording the SF ID in Postgres, the retry will re-upload, creating an orphaned ContentVersion. This is an accepted low-frequency edge case. A scheduled SF batch job can clean up orphaned ContentVersions (those not linked to any `Messenger_Attachment__c` record) weekly.

### 1.10 Inbound DTO Sender (`internal/salesforce/inbound_sender.go`)

Send the normalized DTO to the new SF Apex REST endpoint.

```
POST https://{instance_url}/services/apexrest/tgint/gateway-inbound
Content-Type: application/json
Authorization: Bearer {access_token}

{
  "channelId": "a1B...",
  "idempotencyKey": "uuid-from-postgres-inbound-messages-id",
  "messages": [
    {
      "chatExternalId": "-1001234567890",
      "chatType": "supergroup",
      "chatTitle": "My Group",
      "messageExternalId": "42",
      "senderExternalId": "987654321",
      "senderName": "John Doe",
      "messageText": "Hello world",
      "messageType": "photo",
      "sentAt": "2024-01-15T10:30:00.000Z",
      "replyToExternalId": "41",
      "isEdited": false,
      "attachments": [
        {
          "mediaExternalId": "large_id",
          "mimeType": "image/jpeg",
          "fileName": null,
          "fileSize": 45678,
          "sortOrder": 1,
          "width": 800,
          "height": 600,
          "contentVersionId": "068xxxxxxxxxxxx"
        }
      ]
    }
  ]
}
```

**Authentication:** The Go Gateway authenticates to SF via JWT Bearer OAuth2 (see section 1.8). The `Authorization: Bearer {access_token}` header is set by `doRequest()`. The `X-Gateway-Secret` header is **NOT used** for this endpoint — the OAuth2 token provides sufficient authentication. The `X-Gateway-Secret` is only used for the `/api/channel-config` endpoint (Go-side) where SF pushes config to Go.

Response from SF:
```json
{
  "success": true,
  "messageIds": ["a2C..."]
}
```

Error responses: 400 (bad request), 401 (auth failed), 404 (channel not found), 410 (channel inactive), 500 (server error).

**All error responses include JSON body** for consistent Go-side parsing:
```json
{"success": false, "error": "Channel not found", "messageIds": []}
```

```go
type InboundSendResult struct {
    Success    bool
    MessageIDs []string // SF Messenger_Message__c IDs
    Error      string
}

// SendInbound posts normalized DTOs to SF. Returns SF message IDs on success.
func (c *SalesforceClient) SendInbound(ctx context.Context, channelID string, idempotencyKey string, dtos []models.NormalizedInboundDTO) (*InboundSendResult, error)
```

### 1.11 Webhook Handler (`internal/handlers/webhook.go`)

Main HTTP handler orchestrating the full inbound flow.

```go
type WebhookHandler struct {
    repo       *persistence.MessageRepository
    parser     *telegram.Parser
    configRepo *persistence.ChannelConfigRepository
    workerPool *worker.Pool
    logger     *slog.Logger
}

func (h *WebhookHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 0. Apply request body size limit
    r.Body = http.MaxBytesReader(w, r.Body, 64*1024) // 64 KB max (Telegram payloads are small)

    // 1. Extract channelType and channelId from URL path
    // 2. Load channel config from Postgres (decrypt config blob)
    // 3. Validate secret token header (constant-time compare via crypto/subtle)
    //    -> 401 if invalid
    // 4. Read request body
    // 5. Parse Telegram JSON -> NormalizedInboundDTO
    //    -> 200 if empty (Telegram sends empty updates sometimes)
    // 6. Persist to Postgres in a single transaction:
    //    SaveInboundWithAttachments(raw_payload encrypted, normalized_dto encrypted, status: pending)
    //    -> If Postgres INSERT fails, return 500 to Telegram (Telegram will retry)
    //    -> This is the ONLY case where we return non-200 to Telegram
    // 7. Return 200 to Telegram immediately
    // 8. Enqueue message ID to the bounded worker pool (non-blocking)
    //    -> If pool queue is full, message stays as "pending" and retry worker picks it up
    //    NOTE: Steps 9-12 run in a worker goroutine, NOT the request goroutine
    // --- Worker goroutine (uses context.Background() with 5-minute timeout, NOT r.Context()) ---
    // 9. SetProcessing(messageID) — atomically claim the message (SELECT FOR UPDATE SKIP LOCKED)
    //    -> If already claimed, skip (another worker/retry got it)
    // 10. For each attachment:
    //     a. Check fileSize from DTO — if > 10 MB, SkipAttachment("file too large"), continue
    //     b. Download from Telegram (getFile + download with io.LimitReader)
    //     c. Store file body in Postgres (encrypted)
    //     d. Upload to SF ContentVersion (multipart/form-data)
    //     e. Store SF ContentVersion ID in Postgres, clear file_body
    //     f. Set contentVersionId on DTO attachment
    // 11. Send normalized DTO to SF Apex REST
    // 12. On success: MarkDelivered(messageID, sfMessageID)
    //     On failure: MarkFailed(messageID, error) — sets retry_after based on backoff schedule
}
```

**CRITICAL requirements:**

1. **Return 200 to Telegram BEFORE processing.** Telegram expects fast webhook responses (< 60s) and will retry/disable webhook if responses are slow.
2. **Return 500 if Postgres persist fails.** Telegram will retry the webhook. This is the only case we return non-200.
3. **Worker goroutine must use `context.WithTimeout(context.Background(), 5*time.Minute)`.** The request context `r.Context()` is cancelled when the handler returns (after responding 200). Using it in the goroutine will cause all context-aware calls to fail with `context.Canceled`.
4. **Enqueue to worker pool, not raw `go func()`.** The worker pool bounds concurrency (see section 1.11.1). If the pool is full, the message stays "pending" and the retry worker picks it up.

**Rate limiting (REQUIRED):**

Apply per-IP rate limiting on the webhook endpoint:
```go
// In main.go, wrap the webhook handler:
limiter := rate.NewLimiter(rate.Limit(100), 200) // 100 req/s sustained, 200 burst
// Or use per-IP middleware with golang.org/x/time/rate
```

Consider adding Telegram IP allowlisting (Telegram publishes their server IP ranges at `https://core.telegram.org/bots/webhooks#the-short-version`), but do NOT hard-code IPs — they change. Log warnings for requests from non-Telegram IPs.

### 1.11.1 Bounded Worker Pool (`internal/worker/pool.go`)

```go
type Pool struct {
    tasks    chan func()
    wg       sync.WaitGroup
    logger   *slog.Logger
}

// NewPool creates a bounded worker pool with maxWorkers goroutines.
// Workers process tasks from the channel. The pool integrates with graceful shutdown.
func NewPool(maxWorkers int, queueSize int, logger *slog.Logger) *Pool

// Submit enqueues a task. Non-blocking: returns false if queue is full
// (caller should leave message as "pending" for retry worker).
func (p *Pool) Submit(task func()) bool

// Shutdown signals workers to stop and waits for in-flight tasks to complete.
// Called during graceful shutdown. Blocks until all workers finish or ctx expires.
func (p *Pool) Shutdown(ctx context.Context)
```

**Implementation:** Launch `maxWorkers` goroutines in `NewPool`, each reading from the `tasks` channel. `wg.Add(1)` before task execution, `wg.Done()` after. `Shutdown()` closes the channel and calls `wg.Wait()`.

### 1.12 Retry Worker (`internal/retry/worker.go`)

Background goroutine that picks up failed messages and retries the full pipeline.

```go
// StartRetryWorker runs every 30 seconds, picks up eligible messages
// (status=failed, retry_after <= now, attempt_count < 10).
// Uses SELECT FOR UPDATE SKIP LOCKED to prevent conflicts with worker pool.
// Submits retryable messages to the worker pool for processing.
// Max 10 attempts (see backoff schedule in section 1.7).
// After 10 failures, message moves to "dead" status.
func StartRetryWorker(ctx context.Context, repo *MessageRepository, pool *worker.Pool, interval time.Duration, logger *slog.Logger)
```

**Retry worker handles the FULL pipeline per attachment status:**
1. If attachment status = `pending` -> resolve `file_id` via `getFile`, download, then upload to SF
2. If attachment status = `downloaded` -> upload to SF
3. If attachment status = `uploaded` -> proceed to DTO send
4. If attachment status = `skipped` or `failed` -> skip this attachment

This ensures that if Go crashes at any point during processing, the retry worker resumes from the correct step.

**Circuit breaker for SF outages:**

If 5 consecutive SF calls fail (across any messages), enter "circuit open" state:
- Stop submitting retry tasks to the worker pool
- Every 60 seconds, attempt a single SF health check (`GET /services/data/v62.0/`)
- When health check succeeds, close the circuit and resume retries
- Log at ERROR level when circuit opens, INFO when it closes

### 1.13 Entry Point (`cmd/messenger/main.go`)

```go
func main() {
    // 1. Load config from env (panic on missing required values)
    // 2. Initialize structured logger: slog.New(slog.NewJSONHandler(os.Stdout, &opts))
    //    - All log entries include: "service": "messageforge-gateway"
    //    - Request-scoped logs include: "request_id": uuid, "channel_id": id
    // 3. Connect to Postgres via pgxpool with connection limits (see section 1.6)
    // 4. Run migrations via golang-migrate (embedded SQL files)
    // 5. Initialize encryption with ENCRYPTION_KEYS (parse key versions)
    // 6. Initialize SF JWT Bearer OAuth2 client
    // 7. Initialize repositories (MessageRepository, ChannelConfigRepository)
    // 8. Initialize Telegram parser + downloader (shared http.Client)
    // 9. Initialize bounded worker pool (maxWorkers from config, queue size = maxWorkers * 2)
    // 10. Register HTTP routes:
    //     POST /webhook/{channelType}/{channelId} -> WebhookHandler (with rate limiter middleware)
    //     GET  /webhook/{channelType}/{channelId} -> VerifyHandler (for Meta channels, tech debt)
    //     GET  /health -> HealthHandler (checks DB ping + SF token validity)
    //     POST /api/channel-config -> ConfigHandler (receives channel config from SF, validated by X-Gateway-Secret)
    //     GET  /api/admin/queue-stats -> QueueStatsHandler (protected by X-Gateway-Secret, returns pending/failed/dead counts)
    // 11. Start retry worker goroutine
    // 12. Start HTTP server
    // 13. Graceful shutdown on SIGTERM/SIGINT:
    //     a. server.Shutdown(ctx) — stop accepting new connections, drain active handlers
    //     b. workerPool.Shutdown(ctx) — wait for in-flight worker goroutines (30s timeout)
    //     c. retryWorker cancel via context
    //     d. Close Postgres pool
    //     e. Log "shutdown complete"
}
```

Use `net/http` + `http.ServeMux` (Go 1.22+ has path params). No external router needed.

**Structured logging requirements (slog):**

Every log entry must be structured JSON for Fly.io aggregation:
```json
{"time":"2024-01-15T10:30:00Z","level":"INFO","msg":"webhook received","request_id":"uuid","channel_id":"a1B...","channel_type":"Telegram_Bot"}
```

**Correlation ID:** The Postgres `inbound_messages.id` UUID serves as the correlation ID. Attach it to all log entries for a given message's lifecycle (receive -> persist -> download -> upload -> send -> delivered/failed).

### 1.14 Docker & Deployment

**docker-compose.yml** (local dev):
```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: messenger
      POSTGRES_PASSWORD: localdev
      POSTGRES_DB: messenger
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

**Dockerfile:**
```dockerfile
FROM golang:1.23.8-bookworm AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /messenger ./cmd/messenger

FROM gcr.io/distroless/static-debian12
COPY --from=build /messenger /messenger
EXPOSE 8080
ENTRYPOINT ["/messenger"]
```

> Go builder image pinned to `1.23.8-bookworm` for reproducible builds. Update periodically for security patches.

**fly.toml:**
```toml
app = "messageforge-backend"
primary_region = "ams"

[build]

[http_service]
  internal_port = 8080
  force_https = true

[env]
  LOG_LEVEL = "info"
  MAX_WORKERS = "20"
```

Secrets set via: `fly secrets set SALESFORCE_LOGIN_URL=... SALESFORCE_CLIENT_ID=... SALESFORCE_USERNAME=... SALESFORCE_PRIVATE_KEY=... DATABASE_URL=... ENCRYPTION_KEYS=... GATEWAY_SECRET=...`

**Postgres backup strategy:** Configure `pg_dump` via Fly.io machines cron or use Fly.io volume snapshots. Daily backups with 7-day retention. Document the backup restoration procedure.

**Deployment note — single region:** The Fly.io app runs in `ams` (Amsterdam). If the SF org is in a US data center, expect cross-Atlantic latency on SF API calls (~100-200ms per call). This is acceptable for Phase 2 with a single customer. For multi-region in the future, deploy Go Gateway in the same region as the SF org.

### 1.15 `.env.example`

```
PORT=8080
LOG_LEVEL=debug
MAX_WORKERS=20

# Salesforce Connected App (JWT Bearer Flow)
SALESFORCE_LOGIN_URL=https://login.salesforce.com
SALESFORCE_CLIENT_ID=3MVG9...
SALESFORCE_USERNAME=api.user@myorg.com
# PEM-encoded RSA private key (escaped newlines or file path)
SALESFORCE_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\nMIIE...\n-----END RSA PRIVATE KEY-----"

# PostgreSQL
DATABASE_URL=postgres://messenger:localdev@localhost:5432/messenger?sslmode=disable

# Encryption keys (comma-separated, format: version:base64key, first = active)
ENCRYPTION_KEYS=v1:BASE64_ENCODED_32_BYTE_KEY

# Gateway-SF shared secret (256-bit random, hex-encoded)
# Generate with: openssl rand -hex 32
GATEWAY_SECRET=...
```

### 1.16 Observability

**Metrics endpoint** (`GET /metrics`, Prometheus-compatible):

| Metric | Type | Description |
|--------|------|-------------|
| `gateway_webhook_received_total` | Counter | Webhooks received, labeled by `channel_type`, `status_code` |
| `gateway_sf_delivery_total` | Counter | SF deliveries, labeled by `result` (success/failure) |
| `gateway_sf_delivery_duration_seconds` | Histogram | SF delivery latency |
| `gateway_queue_depth` | Gauge | Messages by status (pending/processing/failed/dead) |
| `gateway_attachment_upload_total` | Counter | ContentVersion uploads, labeled by `result` |
| `gateway_retry_total` | Counter | Retry attempts |
| `gateway_circuit_breaker_state` | Gauge | 0=closed, 1=open |

**Health check** (`GET /health`):

Returns 200 if all checks pass, 503 if any fail:
```json
{
  "status": "healthy",
  "checks": {
    "postgres": {"status": "ok", "latency_ms": 2},
    "salesforce": {"status": "ok", "token_age_minutes": 45},
    "worker_pool": {"status": "ok", "active_workers": 3, "queue_depth": 0}
  }
}
```

**Admin queue stats** (`GET /api/admin/queue-stats`, protected by `X-Gateway-Secret`):

```json
{
  "pending": 0,
  "processing": 2,
  "failed": 1,
  "dead": 0,
  "delivered_last_hour": 47,
  "last_delivery_at": "2024-01-15T10:30:00Z"
}
```

---

## Work Stream 2: Salesforce - New Gateway Inbound Endpoint

### 2.1 New Apex REST Class: `GatewayInboundAPI.cls`

Replaces `MessengerInboundAPI.cls` for gateway-sourced messages. The key difference: webhook secret validation already done by Go, files already uploaded as ContentVersion.

```apex
@RestResource(urlMapping='/tgint/gateway-inbound')
global with sharing class GatewayInboundAPI {

    @HttpPost
    global static void doPost() {
        RestRequest req  = RestContext.request;
        RestResponse res = RestContext.response;

        // 1. Parse request body
        GatewayInboundRequest payload = (GatewayInboundRequest)
            JSON.deserialize(req.requestBody.toString(), GatewayInboundRequest.class);

        // 2. Load channel
        Messenger_Channel__c channel = loadChannel(payload.channelId);
        if (channel == null) {
            res.statusCode = 404;
            res.responseBody = Blob.valueOf('{"success":false,"error":"Channel not found","messageIds":[]}');
            return;
        }
        if (!channel.Active__c || channel.Is_Compromised__c) {
            res.statusCode = 410;
            res.responseBody = Blob.valueOf('{"success":false,"error":"Channel inactive or compromised","messageIds":[]}');
            return;
        }

        // 3. Convert gateway DTOs to NormalizedInboundDTO
        //    CRITICAL: Set contentVersionId on NormalizedAttachmentDTO during conversion
        //    CRITICAL: Parse sentAt ISO 8601 string using:
        //      dto.sentAt = (Datetime) JSON.deserialize('"' + msg.sentAt + '"', Datetime.class);
        //    DO NOT use Datetime.valueOf() — it will fail on the 'T' and 'Z' characters.
        List<NormalizedInboundDTO> dtos = convertDTOs(payload.messages);

        // 4. Reuse InboundDataService pipeline (chat upsert, message insert, PE publish)
        //    Pass skipMediaDownload=true to prevent MediaDownloadQueueable from being enqueued
        new InboundDataService().processInboundMessages(dtos, channel, true);

        // 5. Return success with SF message IDs
        //    (message IDs are set on the DTO objects by processInboundMessages)
        res.statusCode = 200;
        res.responseBody = Blob.valueOf(JSON.serialize(
            new GatewayInboundResponse(true, getMessageIds(dtos))
        ));
    }
}
```

**Authentication:** The Go Gateway authenticates via OAuth2 JWT Bearer flow. The API user's access token is validated by the Salesforce platform automatically on every REST API call. No `X-Gateway-Secret` header validation is needed in this endpoint — the OAuth2 token is sufficient.

**Why no X-Gateway-Secret here:** The gateway secret is for authenticating SF -> Go pushes (channel config). For Go -> SF calls, the OAuth2 access token provides stronger authentication with built-in token expiry, scope restrictions, and audit trail.

**`with sharing` on outer class:** Correct. The Go Gateway authenticates as a dedicated API User (Connected App). The `InboundDataService` uses `without sharing` internally (see section 2.4) to bypass OWD restrictions for system-level operations.

**Error response format:** All error responses include a JSON body with `success`, `error`, and `messageIds` fields for consistent Go-side parsing. The Go client checks HTTP status code first, then parses the body.

**ContentVersion linking is handled at insert time** (see section 2.4). The `linkContentVersions` step is eliminated — `processInboundMessages` sets `Media_URL__c` directly when `contentVersionId` is present on the attachment DTO.

### 2.2 Request/Response DTOs

```apex
public class GatewayInboundRequest {
    public String channelId;
    public String idempotencyKey;  // Go-side inbound_messages.id UUID for deduplication
    public List<GatewayMessageDTO> messages;
}

public class GatewayMessageDTO {
    public String chatExternalId;
    public String chatType;
    public String chatTitle;
    public String messageExternalId;
    public String senderExternalId;
    public String senderName;
    public String messageText;
    public String messageType;
    public String sentAt;           // ISO 8601
    public String replyToExternalId;
    public Boolean isEdited;
    public List<GatewayAttachmentDTO> attachments;
}

public class GatewayAttachmentDTO {
    public String mediaExternalId;
    public String mimeType;
    public String fileName;
    public Long fileSize;
    public Integer sortOrder;
    public Integer durationSeconds;
    public Integer width;
    public Integer height;
    public String contentVersionId;  // SF ContentVersion ID (already uploaded by Go)
}

public class GatewayInboundResponse {
    public Boolean success;
    public List<String> messageIds;
    public String error;

    public GatewayInboundResponse(Boolean success, List<String> messageIds) {
        this.success = success;
        this.messageIds = messageIds;
    }
}
```

### 2.3 New Custom Metadata Fields

**`Gateway_Secret__c`** on `Messenger_Settings__mdt`:

```xml
<!-- objects/Messenger_Settings__mdt/fields/Gateway_Secret__c.field-meta.xml -->
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Gateway_Secret__c</fullName>
    <description>Encrypted shared secret for authenticating between SF and Go Gateway.
        Store the encrypted value (output of EncryptionService.encrypt(plaintext_secret)).
        DO NOT store the plaintext secret directly.</description>
    <fieldManageability>DeveloperControlled</fieldManageability>
    <label>Gateway Secret (Encrypted)</label>
    <length>500</length>
    <type>Text</type>
</CustomField>
```

**Storage and validation flow (CRITICAL — resolves the encrypt/decrypt contradiction):**

1. **At setup time:** Admin enters the plaintext gateway secret in the Channel Setup UI. The UI calls `EncryptionService.encrypt(plaintextSecret)` and stores the encrypted value in `Gateway_Secret__c`.
2. **At validation time (SF -> Go config push):** `EncryptionService.decrypt(settings.Gateway_Secret__c)` recovers the plaintext, which is sent as the `X-Gateway-Secret` header to Go.
3. **The Go Gateway** stores the same plaintext secret in `GATEWAY_SECRET` env var and compares incoming headers against it.
4. **The value in the metadata record is ALWAYS the encrypted form.** The plaintext NEVER appears in Custom Metadata XML, version control, or Setup UI.

> **Note on Custom Metadata visibility:** Custom Metadata Text fields are readable by admins and appear in metadata retrieves. However, since we store the encrypted ciphertext (not the plaintext), admin visibility only exposes the ciphertext. Anyone who can decrypt it already has access to `Encryption_Key__mdt`, making the security model symmetric with the existing `EncryptionService` pattern. This is a known and accepted limitation.

**`Gateway_URL__c`** on `Messenger_Settings__mdt`:

```xml
<!-- objects/Messenger_Settings__mdt/fields/Gateway_URL__c.field-meta.xml -->
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Gateway_URL__c</fullName>
    <description>Go Gateway base URL for webhook registration.
        Example: https://messageforge-backend.fly.dev</description>
    <fieldManageability>DeveloperControlled</fieldManageability>
    <label>Gateway URL</label>
    <type>Url</type>
</CustomField>
```

> **Note:** `Go_Middleware_URL__c` already exists on `Messenger_Settings__mdt` from earlier architecture. It should be deprecated (left in place for backward compatibility) and `Gateway_URL__c` used for all new code.

**Metadata file deployment:** Both field XML files must be checked into `force-app/main/default/objects/Messenger_Settings__mdt/fields/` and included in the SFDX source deployment.

### 2.4 InboundDataService Extraction

**Action:** Extract `DataService` inner class from `MessengerInboundAPI.cls` into a standalone `InboundDataService.cls`.

```apex
/**
 * InboundDataService — extracted from MessengerInboundAPI.DataService.
 *
 * Sharing: WITHOUT SHARING.
 * Justification: This service runs in two contexts:
 *   1. Site Guest User (via MessengerInboundAPI) — Guest User has no CRUD permissions.
 *   2. Connected App API User (via GatewayInboundAPI) — API User may not have Modify All
 *      on namespace objects, and OWD may be Private on Messenger_Chat__c.
 * In both cases, the service must operate system-wide to upsert chats, insert messages,
 * and publish Platform Events regardless of the calling user's sharing context.
 */
public without sharing class InboundDataService {

    /**
     * processInboundMessages — main pipeline.
     *
     * @param dtos List of normalized inbound DTOs to process
     * @param channel The Messenger_Channel__c record for this channel
     * @param skipMediaDownload If true, do NOT enqueue MediaDownloadQueueable.
     *        Set to true when called from GatewayInboundAPI (files already uploaded by Go).
     *        Set to false when called from MessengerInboundAPI (legacy path, files need download).
     *
     * Pipeline:
     * 1. Deduplicate: Query existing Messenger_Message__c by Message_External_ID__c within
     *    the chat. Skip messages that already exist (prevents duplicate delivery on Go retry).
     * 2. loadExistingChats() — SOQL
     * 3. matchLeadContact() — SOQL (phone channels only, skipped for Telegram)
     * 4. buildAndUpsertChats() — DML 1
     * 5. insertMessages() — DML 2
     * 6. insertAttachments() — DML 3
     *    CRITICAL: When contentVersionId is set on a NormalizedAttachmentDTO, set:
     *      att.Media_URL__c = '/sfc/servlet.shepherd/version/download/' + contentVersionId
     *      att.Content_Version_ID__c = contentVersionId
     *    This happens at INSERT time (single DML), NOT as a separate update loop.
     *    This eliminates the old linkContentVersions() DML-in-loop anti-pattern.
     * 7. publishInboundEvents() — DML 4+
     *    CRITICAL: Chunk Platform Events to 150 per EventBus.publish() call.
     *    Salesforce limits Platform Events to 150 per transaction.
     *    Implementation:
     *      List<Database.SaveResult> allResults = new List<Database.SaveResult>();
     *      for (Integer i = 0; i < events.size(); i += 150) {
     *          List<SObject> chunk = new List<SObject>();
     *          for (Integer j = i; j < Math.min(i + 150, events.size()); j++) {
     *              chunk.add(events[j]);
     *          }
     *          allResults.addAll(EventBus.publish(chunk));
     *      }
     * 8. If !skipMediaDownload AND attachments exist:
     *      System.enqueueJob(new MediaDownloadQueueable(...))
     */
    public void processInboundMessages(
        List<NormalizedInboundDTO> dtos,
        Messenger_Channel__c channel,
        Boolean skipMediaDownload
    ) { ... }
}
```

**Message deduplication (prevents duplicate delivery on Go retry):**

Before inserting messages, query existing `Messenger_Message__c` records:
```apex
// Collect all messageExternalIds from the DTOs
Set<String> externalIds = new Set<String>();
for (NormalizedInboundDTO dto : dtos) {
    externalIds.add(dto.messageExternalId);
}

// Query existing messages in these chats with matching external IDs
Map<String, Id> existingMessages = new Map<String, Id>();
for (Messenger_Message__c msg : [
    SELECT Id, Message_External_ID__c
    FROM Messenger_Message__c
    WHERE Message_External_ID__c IN :externalIds
    AND Messenger_Chat__r.Chat_External_ID__c IN :chatExternalIds
]) {
    existingMessages.put(msg.Message_External_ID__c, msg.Id);
}

// Filter out already-existing messages (standard Apex for-loop — Apex has no stream API)
List<NormalizedInboundDTO> filtered = new List<NormalizedInboundDTO>();
for (NormalizedInboundDTO dto : dtos) {
    if (!existingMessages.containsKey(dto.messageExternalId)) {
        filtered.add(dto);
    }
}
dtos = filtered;
```

This adds 1 SOQL query but prevents duplicate messages when Go retries after a timeout (SF processed successfully but response was lost).

**NormalizedAttachmentDTO — add `contentVersionId` field:**

The existing `NormalizedAttachmentDTO.cls` must be updated to include:
```apex
public String contentVersionId;  // SF ContentVersion ID (set by Go Gateway, null for legacy path)
```

**Messenger_Attachment__c — add `Content_Version_ID__c` field:**

Create a new custom field on `Messenger_Attachment__c`:
```xml
<!-- objects/Messenger_Attachment__c/fields/Content_Version_ID__c.field-meta.xml -->
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Content_Version_ID__c</fullName>
    <description>Salesforce ContentVersion ID for files uploaded by Go Gateway</description>
    <externalId>false</externalId>
    <label>Content Version ID</label>
    <length>18</length>
    <type>Text</type>
</CustomField>
```

**Backward compatibility:** `MessengerInboundAPI.cls` continues to call `new InboundDataService().processInboundMessages(dtos, channel, false)` with `skipMediaDownload = false`. The legacy path is unchanged.

**ChannelAccessService.grantAccessToChat():** Verify that this is called via an after-insert trigger on `Messenger_Chat__c`. If it is trigger-based, the gateway path works automatically (the `buildAndUpsertChats` DML fires the trigger). If it is only called from `DataService` directly, add the call to `InboundDataService.processInboundMessages()` after chat upsert. Check the trigger handlers before implementing.

### 2.5 Test Class: `GatewayInboundAPITest.cls`

**Test infrastructure requirements:**
- `@IsTest` class with `@TestSetup` method for shared test data
- A `TestDataFactory` helper (or inline) to create `Messenger_Channel__c`, `Messenger_Settings__mdt` (use `Test.createData`)
- `@SeeAllData=false` (mandatory for AppExchange)
- `Test.startTest()` / `Test.stopTest()` wrapping governor-limit-sensitive operations

**Test coverage targets (ALL required for AppExchange security review):**

| Test | Expected | Notes |
|------|----------|-------|
| Valid request -> 200, message created | Assert `Messenger_Message__c` inserted | Happy path |
| Missing channel -> 404 | Assert JSON error response | |
| Inactive channel -> 410 | Assert JSON error response | Set `Active__c = false` |
| Compromised channel -> 410 | Assert JSON error response | Set `Is_Compromised__c = true` |
| Text message (no attachments) | Chat upserted, message inserted, PE published | |
| Photo with ContentVersionId | `Messenger_Attachment__c.Content_Version_ID__c` set, `Media_URL__c` set | |
| Duplicate message (same messageExternalId) | Second call should not create duplicate | Deduplication test |
| Multiple messages in batch (5) | All processed, all message IDs returned | |
| **Bulk: 200 messages** | All 200 processed, PE chunked at 150 | **AppExchange requires 200-record tests** |
| Bulk: 200 messages with 1 attachment each | 200 messages + 200 attachments, no governor limit violation | |
| ISO 8601 sentAt parsing | `Datetime` field correctly populated | Edge case: `"2024-01-15T10:30:00.000Z"` |
| isEdited = true | Message inserted with `Is_Edited__c = true` | |
| InboundDataService: both callers work | Call from legacy `MessengerInboundAPI` (skipMediaDownload=false) and `GatewayInboundAPI` (true) | Extraction test |
| skipMediaDownload = false | `MediaDownloadQueueable` enqueued | Legacy path |
| skipMediaDownload = true | `MediaDownloadQueueable` NOT enqueued | Gateway path |

### 2.6 Connected App as SFDX Metadata

The Connected App for Go Gateway should be declared as SFDX metadata for consistent deployment:

```
force-app/main/default/connectedApps/MessageForge_Gateway.connectedApp-meta.xml
```

```xml
<ConnectedApp xmlns="http://soap.sforce.com/2006/04/metadata">
    <contactEmail>admin@messageforge.com</contactEmail>
    <label>MessageForge Gateway</label>
    <oauthConfig>
        <callbackUrl>https://login.salesforce.com/services/oauth2/callback</callbackUrl>
        <certificate><!-- X.509 certificate for JWT Bearer flow --></certificate>
        <consumerKey><!-- populated at deploy time --></consumerKey>
        <isAdminApproved>true</isAdminApproved>
        <scopes>Api</scopes>
    </oauthConfig>
    <oauthPolicy>
        <ipRelaxation>ENFORCE</ipRelaxation>
        <refreshTokenPolicy>SPECIFIC_LIFETIME</refreshTokenPolicy>
    </oauthPolicy>
</ConnectedApp>
```

> **Note:** The `certificate` field contains the X.509 public certificate (PEM). The corresponding RSA private key is stored only in Fly.io secrets (`SALESFORCE_PRIVATE_KEY`). The private key NEVER touches Salesforce.

**API User Permission Set:**

Create a Permission Set in SFDX metadata:

```
force-app/main/default/permissionsets/MessageForge_Gateway_API.permissionset-meta.xml
```

Grants:
- CRUD on `tgint__Messenger_Chat__c`, `tgint__Messenger_Message__c`, `tgint__Messenger_Attachment__c`
- Read on `tgint__Messenger_Channel__c`
- Create on `tgint__Inbound_Message__e` (Platform Event)
- Create on `ContentVersion`
- API Enabled
- Apex REST access to `GatewayInboundAPI`

---

## Work Stream 3: Webhook Registration Update

### 3.1 Update `TelegramWebhookService.cls`

Change webhook URL from SF Site to Go Gateway:

```apex
// OLD (in ChannelSetupController.registerWebhook):
// webhookUrl = siteBase + '/services/apexrest/tgint/webhook/telegram_bot/' + channelId

// NEW:
// webhookUrl = gatewayBase + '/webhook/telegram_bot/' + channelId
```

The `gatewayBase` comes from `Messenger_Settings__mdt.Gateway_URL__c` (new field, e.g. `https://messageforge-backend.fly.dev`).

### 3.2 Push Channel Config to Go Gateway (with sync, update, and delete)

When a Telegram channel is created/activated, SF must push the channel config (bot token, webhook secret) to Go Gateway so Go can validate webhooks and download files.

**Config push endpoint (Go-side):**

```
POST https://messageforge-backend.fly.dev/api/channel-config
X-Gateway-Secret: {decrypted_gateway_secret}
Content-Type: application/json

{
  "channelId": "a1B...",
  "channelType": "Telegram_Bot",
  "botToken": "123456:ABC-DEF...",
  "webhookSecret": "random-uuid-secret",
  "isActive": true
}
```

Go Gateway receives this and stores in `channel_configs` table (encrypted).

**Go handler for this:** `POST /api/channel-config` in `internal/handlers/config.go`. Validates `X-Gateway-Secret` header. Upserts `channel_configs` record by `channel_id`.

**Channel config sync — COMPLETE lifecycle (not just creation):**

| Event | SF Action | Go Gateway Action |
|-------|-----------|-------------------|
| Channel created + activated | Push config to Go (callout) | Upsert `channel_configs` |
| Channel deactivated | Push config with `isActive: false` | Set `is_active = false`, reject webhooks |
| Channel reactivated | Push config with `isActive: true` | Set `is_active = true`, resume processing |
| Bot token rotated | Push updated config (callout) | Upsert `channel_configs` with new encrypted token |
| Channel soft-deleted (suspended) | Push config with `isActive: false` | Set `is_active = false` |
| Channel hard-deleted | Send `DELETE /api/channel-config/{channelId}` | Delete `channel_configs` row |

**Go delete endpoint:** `DELETE /api/channel-config/{channelId}` in `internal/handlers/config.go`. Validates `X-Gateway-Secret`.

**Config push failure rollback:**

If the config push to Go fails after webhook registration with Telegram succeeds, the webhook is active but Go has no config to validate it — all webhooks will be rejected with 401.

**Rollback strategy:**
1. Register webhook with Telegram (callout #1)
2. Push config to Go (callout #2)
3. If step 2 fails, call Telegram `deleteWebhook` to unregister (callout #3)
4. Throw exception to the user: "Channel setup failed: could not sync config with Gateway. Please try again."

> **Note on Apex callout ordering:** All three callouts happen BEFORE any DML. The existing `ChannelSetupController.registerWebhook()` already follows this pattern (callout first, DML second). The new config push callout is added between webhook registration and DML.

**Admin reconciliation endpoint (Go-side):**

`POST /api/channel-config/sync` — accepts a full list of active channel configs from SF and upserts all of them. Used for bulk reconciliation (e.g., after Go database restore). Protected by `X-Gateway-Secret`.

### 3.3 New Custom Metadata Field

See section 2.3 for `Gateway_URL__c`.

### 3.4 Update `ChannelSetupController.cls`

Replace `getSiteBaseUrl()`:

```apex
private static String getGatewayBaseUrl() {
    List<Messenger_Settings__mdt> settings = [
        SELECT Gateway_URL__c
        FROM Messenger_Settings__mdt
        WHERE Is_Active__c = true
        LIMIT 1
    ];
    if (!settings.isEmpty() && String.isNotBlank(settings[0].Gateway_URL__c)) {
        return settings[0].Gateway_URL__c;
    }
    throw new ConfigException('Gateway URL not configured in Messenger Settings');
}
```

Update `registerWebhook()` to:
1. Build webhook URL using gateway base
2. Register webhook with Telegram (callout #1)
3. Push channel config to Go Gateway (callout #2)
4. If config push fails, call `deleteWebhook` on Telegram (callout #3, rollback)
5. Store webhook secret (DML — AFTER all callouts)
6. Insert audit log (DML — AFTER all callouts)

**Config push on update/delete:**

Add a `pushChannelConfigToGateway` method (NOT `@future`) that is called from:
- `registerWebhook()` — after successful Telegram registration
- `toggleActive()` — when channel is activated/deactivated
- `softDeleteChannel()` — push `isActive: false`
- `hardDeleteChannel()` — send DELETE request

> **Callout after DML concern:** `toggleActive()` and `softDeleteChannel()` perform DML before the config push would occur. Salesforce prohibits callouts after uncommitted DML in the same transaction. Solution: Use `@future(callout=true)` for the config push in these methods. Only `registerWebhook()` can do inline callout (it does all callouts before DML).

```apex
@future(callout=true)
private static void pushChannelConfigAsync(String channelId, String channelType, Boolean isActive) {
    // Load bot token, webhook secret from channel config
    // Decrypt gateway secret from Messenger_Settings__mdt
    // POST to Gateway URL + '/api/channel-config'
}

@future(callout=true)
private static void deleteChannelConfigAsync(String channelId) {
    // Decrypt gateway secret from Messenger_Settings__mdt
    // DELETE to Gateway URL + '/api/channel-config/' + channelId
}
```

**Pre-existing SOQL injection note:** `ChannelSetupController.getConfigToken()` (around line 653) uses string concatenation for a `channelId` parameter in dynamic SOQL. While the value is a Salesforce record Id in practice, the parameter type is `String`. When modifying this class, replace string concatenation with bind variables:

```apex
// BEFORE (injection risk):
// 'WHERE Channel__c = \'' + channelId + '\''

// AFTER (safe):
List<SObject> configs = Database.query(
    'SELECT ' + tokenField + ' FROM ' + configObj
    + ' WHERE Channel__c = :channelId LIMIT 1'
);
```

---

## Work Stream 4: LWC Verification

The LWC chat already supports displaying all Telegram media types. Verify these work with ContentVersion URLs:

| Type | Component | How it renders | ContentVersion compatible? |
|------|-----------|---------------|---------------------------|
| text | messengerChatBubble | `<p>` tag | N/A |
| photo | messengerChatBubble | `<img>` 240x200 max | Yes - `/sfc/servlet.shepherd/version/download/{id}` works in `<img src>` |
| video | messengerChatBubble | `<video>` with play overlay | Yes - ContentVersion URL works in `<video src>` |
| voice | messengerChatBubble | `<audio>` with controls | Yes - ContentVersion URL works in `<audio src>` |
| document | messengerChatBubble | File icon + name + download | Yes - download link uses ContentVersion URL |
| sticker | messengerChatBubble | `<img>` 120x120 | Yes - WebP renders in modern browsers |

**Media viewer modal** (`messengerMediaViewer`) already extracts `contentVersionId` from the URL and constructs the download path. No changes needed.

**One thing to verify:** The `Media_URL__c` field must contain the full ContentVersion download path: `/sfc/servlet.shepherd/version/download/{ContentVersionId}`. This is set by `InboundDataService.insertAttachments()` when `contentVersionId` is present on the DTO.

**Session-gated ContentVersion URLs:** The `/sfc/servlet.shepherd/version/download/` URL is session-gated — it requires an authenticated Lightning Experience user. This is correct for the current use case (agents viewing chat in Lightning). If the LWC is ever embedded in an Experience Cloud site with guest access, the ContentVersion sharing model must be reviewed. For Phase 2, guest access is out of scope.

**Explicit test cases for LWC media rendering:**
- [ ] Authenticated user views photo (img tag loads without CORS error)
- [ ] Authenticated user views video (video player loads, playback works)
- [ ] Authenticated user listens to voice note (audio player loads)
- [ ] Authenticated user downloads document (download link works)
- [ ] Authenticated user views sticker (WebP image renders correctly)
- [ ] Content-Type headers are correct (e.g., `image/webp` for stickers)

**No LWC code changes expected.** If issues arise during testing, they'll be CSS/rendering fixes only.

---

## Implementation Order

```
Phase 2.1: Go Gateway Core (5 days)
  ├── Config (env vars, JWT Bearer fields, encryption key rotation)
  ├── Structured logging (slog + JSON handler)
  ├── AES-256-GCM encryption (nonce strategy, key rotation)
  ├── Postgres schema + pgxpool + golang-migrate
  ├── Message repository (transactional save, SELECT FOR UPDATE)
  ├── Telegram parser + tests
  ├── Telegram downloader (io.LimitReader, size check before download)
  ├── Bounded worker pool
  └── HTTP server skeleton (MaxBytesReader, rate limiter)

Phase 2.2: SF Integration (3 days)
  ├── SF JWT Bearer OAuth2 client (compare-and-refresh token pattern)
  ├── ContentVersion upload (multipart/form-data, NOT base64 JSON)
  ├── Inbound DTO sender (idempotency key)
  ├── Retry worker (10 attempts, retry_after column, circuit breaker)
  └── Channel config handler (POST + DELETE endpoints)

Phase 2.3: SF Endpoint (4 days)
  ├── Extract InboundDataService from MessengerInboundAPI (without sharing, skipMediaDownload flag)
  ├── Add contentVersionId to NormalizedAttachmentDTO
  ├── Add Content_Version_ID__c to Messenger_Attachment__c
  ├── Bulkify insertAttachments (set Media_URL__c at insert time)
  ├── Chunk PE publish at 150 per batch
  ├── Add message deduplication by Message_External_ID__c
  ├── GatewayInboundAPI.cls + GatewayInboundAPITest.cls
  ├── Gateway_Secret__c + Gateway_URL__c metadata fields
  ├── Connected App + Permission Set as SFDX metadata
  └── pushChannelConfigToGateway (@future for post-DML calls)

Phase 2.4: Webhook Registration (2 days)
  ├── Update TelegramWebhookService (point to Go Gateway URL)
  ├── Update ChannelSetupController (config push + rollback on failure)
  ├── Config push on activation, deactivation, soft/hard delete
  ├── Fix SOQL injection in getConfigToken (bind variables)
  └── Admin reconciliation endpoint (Go-side)

Phase 2.5: E2E Testing (2 days)
  ├── Local: docker-compose + Go + SF sandbox
  ├── Send test Telegram message -> verify in LWC
  ├── Test all media types (photo, video, voice, document, sticker)
  ├── Test retry (stop SF, send message, restart SF -> message appears)
  ├── Test large file rejection (>10 MB -> graceful skip)
  ├── Test circuit breaker (stop SF for 30 min -> circuit opens, restart -> messages deliver)
  ├── Test duplicate delivery (Go timeout, retry -> no duplicate in SF)
  ├── Test graceful shutdown (deploy Go -> in-flight messages complete or retry)
  └── LWC media rendering tests (all media types load for authenticated user)
```

**Total: ~16 days**

---

## Testing Checklist

### Go Gateway Tests

**Unit tests:**
- [ ] Config: panic on missing required env vars
- [ ] Config: parse ENCRYPTION_KEYS with multiple key versions
- [ ] Telegram parser: text message
- [ ] Telegram parser: photo (selects highest resolution)
- [ ] Telegram parser: document with filename
- [ ] Telegram parser: video with duration
- [ ] Telegram parser: voice note
- [ ] Telegram parser: sticker (WebP)
- [ ] Telegram parser: edited message (isEdited = true)
- [ ] Telegram parser: reply threading
- [ ] Telegram parser: caption on media
- [ ] Telegram parser: empty/malformed JSON (no crash)
- [ ] Webhook auth: valid secret token -> pass
- [ ] Webhook auth: invalid/missing token -> reject
- [ ] Webhook auth: constant-time comparison (no timing leak)
- [ ] Encryption: AES-256-GCM round-trip (encrypt -> decrypt)
- [ ] Encryption: nonce uniqueness (encrypt same plaintext twice -> different ciphertext)
- [ ] Encryption: key rotation (encrypt with v2, decrypt tries v1 first then v2)
- [ ] Encryption: wrong key -> error (not silent corruption)
- [ ] Downloader: resolve file_id -> download URL -> bytes
- [ ] Downloader: file > 10 MB -> skip with error (before download if file_size known)
- [ ] Downloader: io.LimitReader prevents OOM on oversized response
- [ ] Postgres: save message + attachments in single transaction
- [ ] Postgres: SetProcessing with SELECT FOR UPDATE SKIP LOCKED
- [ ] Postgres: save attachments, update SF ID, clear file_body
- [ ] Postgres: SkipAttachment sets status and reason
- [ ] Postgres: mark delivered, mark failed (sets retry_after)
- [ ] Postgres: GetRetryable respects retry_after and attempt_count
- [ ] Postgres: encryption round-trip on raw_payload and normalized_dto
- [ ] SF client: JWT Bearer token acquisition
- [ ] SF client: token refresh on 401 (compare-and-refresh, no thundering herd)
- [ ] SF client: ContentVersion upload via multipart (mock)
- [ ] SF client: inbound DTO send (mock)
- [ ] SF client: API rate limit detection from Sforce-Limit-Info header
- [ ] Worker pool: bounded concurrency (submit > maxWorkers -> queue, not unbounded goroutines)
- [ ] Worker pool: Submit returns false when queue full
- [ ] Worker pool: Shutdown waits for in-flight tasks
- [ ] Webhook handler: full flow (mock SF)
- [ ] Webhook handler: respond 200 before processing
- [ ] Webhook handler: return 500 if Postgres persist fails
- [ ] Webhook handler: MaxBytesReader rejects oversized body
- [ ] Webhook handler: invalid auth -> 401
- [ ] Webhook handler: rate limiter rejects excess requests
- [ ] Retry worker: picks up failed messages, retries full pipeline per attachment status
- [ ] Retry worker: respects retry_after timing
- [ ] Retry worker: circuit breaker opens after 5 consecutive SF failures
- [ ] Retry worker: circuit breaker closes after successful SF health check
- [ ] Config handler: POST upserts channel config (encrypted)
- [ ] Config handler: DELETE removes channel config
- [ ] Config handler: POST /api/channel-config/sync bulk reconciliation upserts all configs
- [ ] Config handler: invalid gateway secret -> 401
- [ ] Health endpoint: returns postgres + SF + worker pool status
- [ ] Queue stats endpoint: returns correct counts by status

**Race condition tests (MUST run with `go test -race ./...`):**
- [ ] Concurrent webhook handlers writing to same channel
- [ ] SF token refresh under concurrent 401s
- [ ] Worker pool submit + shutdown race
- [ ] Retry worker + webhook handler processing same message (should not happen with SKIP LOCKED)

### Salesforce Tests
- [ ] GatewayInboundAPI: valid request -> 200, message created
- [ ] GatewayInboundAPI: missing channel -> 404 with JSON error
- [ ] GatewayInboundAPI: inactive channel -> 410 with JSON error
- [ ] GatewayInboundAPI: compromised channel -> 410 with JSON error
- [ ] GatewayInboundAPI: text message -> chat + message + PE
- [ ] GatewayInboundAPI: photo with ContentVersionId -> Content_Version_ID__c set, Media_URL__c set
- [ ] GatewayInboundAPI: duplicate messageExternalId -> no duplicate message created
- [ ] GatewayInboundAPI: multiple messages in batch (5)
- [ ] GatewayInboundAPI: **bulk 200 messages** (AppExchange required)
- [ ] GatewayInboundAPI: **bulk 200 messages with attachments** (governor limits)
- [ ] GatewayInboundAPI: ISO 8601 sentAt -> Datetime conversion
- [ ] GatewayInboundAPI: isEdited = true -> Is_Edited__c set
- [ ] GatewayInboundAPI: PE chunked at 150 (no LimitException)
- [ ] InboundDataService: extracted correctly, both callers work
- [ ] InboundDataService: skipMediaDownload=false enqueues MediaDownloadQueueable
- [ ] InboundDataService: skipMediaDownload=true does NOT enqueue MediaDownloadQueueable
- [ ] Channel config push: @future callout with encrypted gateway secret
- [ ] Channel config delete: @future callout on hard delete
- [ ] ChannelSetupController: webhook registration -> config push -> DML ordering correct
- [ ] ChannelSetupController: config push failure -> Telegram webhook rolled back

### E2E Tests
- [ ] Send text to Telegram bot -> appears in LWC chat
- [ ] Send photo -> displays as image in bubble, click opens viewer
- [ ] Send document -> shows file icon + name + download link
- [ ] Send video -> video player in bubble, click opens viewer
- [ ] Send voice note -> audio player with duration
- [ ] Send sticker -> WebP image at 120x120
- [ ] Reply to message -> shows reply reference
- [ ] Edit message -> new message with (edited) label in chat
- [ ] Stop SF -> send message -> restart SF -> message appears (retry with circuit breaker)
- [ ] Send 10MB+ file -> graceful skip, text portion still delivered
- [ ] Go Gateway redeploy during message processing -> message eventually delivered (graceful shutdown + retry)
- [ ] Duplicate delivery test: simulate Go timeout -> retry -> no duplicate in SF
- [ ] Channel deactivation -> webhook rejected by Go with 403
- [ ] Channel reactivation -> webhooks resume processing
- [ ] Bot token rotation -> new token pushed to Go, webhooks continue working

---

## SF Connected App Setup (prerequisite)

Before Go Gateway can authenticate to SF:

1. **Create Connected App** in SF Setup (or deploy via SFDX metadata — see section 2.6):
   - OAuth Settings: Enable OAuth
   - Callback URL: `https://login.salesforce.com/services/oauth2/callback`
   - Selected OAuth Scopes: `api` (only — do NOT add `refresh_token`, it is unused with JWT Bearer flow)
   - **Use Digital Signatures:** Upload X.509 certificate (corresponds to the private key in Fly.io)
   - **JWT Bearer flow:** Enabled (check "Enable for Device Flow" is OFF, check "Allow OAuth Username-Password Flows" is OFF)

2. **Generate X.509 certificate and private key:**
   ```bash
   # Generate RSA private key
   openssl genrsa -out server.key 2048
   # Generate self-signed X.509 certificate (valid 10 years)
   openssl req -new -x509 -key server.key -out server.crt -days 3650 -subj "/CN=MessageForge Gateway"
   ```
   - Upload `server.crt` to the Connected App
   - Store `server.key` content in Fly.io: `fly secrets set SALESFORCE_PRIVATE_KEY="$(cat server.key)"`
   - **SECURITY: Delete the local private key file immediately after uploading to Fly.io:** `rm server.key`
     The private key must exist ONLY in Fly.io secrets. Never commit it to version control or leave it on disk.

3. **Create API User** with profile/permission set granting:
   - Assign the `MessageForge_Gateway_API` Permission Set (see section 2.6)
   - API Enabled
   - **Pre-authorize the Connected App** for this user (Setup > Connected Apps > Manage > Edit Policies > Admin approved users are pre-authorized)

4. **Set Fly.io Secrets:**
   ```bash
   fly secrets set \
     SALESFORCE_LOGIN_URL="https://login.salesforce.com" \
     SALESFORCE_CLIENT_ID="3MVG9..." \
     SALESFORCE_USERNAME="api.user@myorg.com" \
     SALESFORCE_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----..." \
     DATABASE_URL="postgres://..." \
     ENCRYPTION_KEYS="v1:BASE64KEY" \
     GATEWAY_SECRET="$(openssl rand -hex 32)"
   ```

5. **Set SF Custom Metadata:**
   - `Messenger_Settings__mdt.Default_Settings`:
     - `Gateway_URL__c` = `https://messageforge-backend.fly.dev`
     - `Gateway_Secret__c` = output of `EncryptionService.encrypt('hex_value_from_GATEWAY_SECRET_env_var')`
       (Run this in Execute Anonymous to get the encrypted value, then set it in the metadata record)

---

## Review Issue Tracker

All issues identified by the Salesforce, Go, and Architecture reviews, with resolution status:

### CRITICAL

| ID | Issue | Resolution |
|----|-------|------------|
| C1 | Username-Password OAuth2 fails AppExchange security review | **FIXED**: Replaced with JWT Bearer flow (sections 1.2, 1.8, 1.15, SF Connected App Setup) |
| C2 | Gateway_Secret__c stores plaintext but plan calls decrypt() | **FIXED**: Store encrypted value, decrypt at comparison time (section 2.3) |
| C3 | AES-256-GCM nonce strategy unspecified | **FIXED**: Random 12-byte nonce per encrypt, key version byte prefix, key rotation via ENCRYPTION_KEYS (section 1.6) |
| C4 | SF token refresh thundering herd / double-refresh race | **FIXED**: Compare-and-refresh pattern with sync.Mutex (section 1.8) |

### HIGH

| ID | Issue | Resolution |
|----|-------|------------|
| H1 | Goroutine leak on graceful shutdown | **FIXED**: Bounded worker pool with sync.WaitGroup, graceful drain (sections 1.11.1, 1.13) |
| H2 | Retry worker double-picks in-flight messages | **FIXED**: SELECT FOR UPDATE SKIP LOCKED, explicit "processing" status (section 1.7) |
| H3 | Background goroutine uses cancelled request context | **FIXED**: context.WithTimeout(context.Background(), 5*time.Minute) (section 1.11) |
| H4 | Platform Event 150-per-transaction limit | **FIXED**: Chunk to 150 per EventBus.publish() (section 2.4) |
| H5 | linkContentVersions is DML-in-a-loop | **FIXED**: Set Media_URL__c at insert time, eliminate separate update (section 2.4) |
| H6 | MediaDownloadQueueable still enqueued for gateway messages | **FIXED**: skipMediaDownload parameter on processInboundMessages (section 2.4) |
| H7 | InboundDataService sharing keyword undefined | **FIXED**: `without sharing` with documented justification (section 2.4) |
| H8 | No message deduplication in SF | **FIXED**: Query existing Message_External_ID__c before insert (section 2.4) |
| H9 | SaveInbound + SaveAttachments not transactional | **FIXED**: SaveInboundWithAttachments in single pgx.Tx (section 1.7) |
| H10 | Webhook body not size-limited | **FIXED**: http.MaxBytesReader 64KB (section 1.11) |
| H11 | pushChannelConfig callout after DML | **FIXED**: @future(callout=true) for post-DML calls, inline for registerWebhook (section 3.4) |

### MEDIUM

| ID | Issue | Resolution |
|----|-------|------------|
| M1 | Retry backoff too aggressive (5 attempts, 13min) | **FIXED**: 10 attempts, ~24h total, circuit breaker (sections 1.7, 1.12) |
| M2 | Channel config sync: no update/delete/rollback | **FIXED**: Full lifecycle sync + rollback on failure (section 3.2) |
| M3 | No rate limiting on webhook endpoint | **FIXED**: Per-IP rate limiter (section 1.11) |
| M4 | Postgres connection pool not configured | **FIXED**: pgxpool with MaxConns=25 (section 1.6) |
| M5 | normalized_dto contains PII, stored unencrypted | **FIXED**: Changed to BYTEA, encrypted (section 1.6) |
| M6 | ISO 8601 sentAt parsing unspecified | **FIXED**: Use JSON.deserialize for Datetime (section 2.1) |
| M7 | Base64 JSON ContentVersion upload inflates size | **FIXED**: Multipart/form-data upload (section 1.9) |
| M8 | 12-20MB file gap: download then reject | **FIXED**: 10MB hard limit checked before download (section 1.5) |
| M9 | No Postgres driver specified, no migration tool | **FIXED**: pgx/v5 + golang-migrate (sections 1.1.1, 1.6) |
| M10 | No structured logging | **FIXED**: slog with JSON handler, correlation IDs (sections 1.1.1, 1.13) |
| M11 | Unbounded goroutine spawning | **FIXED**: Bounded worker pool (section 1.11.1) |
| M12 | SF token refresh thundering herd | **FIXED**: Compare-and-refresh pattern (section 1.8) |
| M13 | Bulk test at 50, AppExchange requires 200 | **FIXED**: 200-message test cases added (section 2.5) |
| M14 | Connected App not as SFDX metadata | **FIXED**: ConnectedApp + PermissionSet metadata (section 2.6) |

### LOW

| ID | Issue | Resolution |
|----|-------|------------|
| L1 | hmac.go / ValidateTelegram misleading naming | **FIXED**: Renamed to webhook_auth.go / ValidateWebhookSecret (sections 1.1, 1.3) |
| L2 | /api/channel-config accepts tokens without IP restriction | **DOCUMENTED**: Accepted risk, protected by gateway secret + HTTPS. Added admin reconciliation endpoint for recovery. |
| L3 | Dockerfile does not pin Go builder version | **FIXED**: Pinned to golang:1.23.8-bookworm (section 1.14) |
| L4 | No observability plan | **FIXED**: Metrics endpoint, deep health check, queue stats (section 1.16) |
| L5 | Backoff values as magic numbers | **FIXED**: Package-level BackoffSchedule constant (section 1.7) |
| L6 | No -race flag in test plan | **FIXED**: Race condition test section added to testing checklist |
| L7 | Edited message SF behavior not specified | **FIXED**: Documented append-only behavior (section 1.4) |
