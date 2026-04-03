# PostgreSQL Schema Reference

PostgreSQL runs as a **dedicated, hosted service** next to the Go middleware (same VPS or managed service like Hetzner Managed PostgreSQL, AWS RDS, DigitalOcean Managed DB).

## Tables

### MTProto Session Storage (UNLOGGED)

```sql
CREATE UNLOGGED TABLE mtproto_sessions (
    id              BIGSERIAL PRIMARY KEY,
    connection_id   BIGINT UNIQUE REFERENCES connections(id) ON DELETE CASCADE,
    session_data    BYTEA NOT NULL,        -- Raw gotd session bytes
    dc_id           INT NOT NULL,
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

**Why UNLOGGED:** Skips Write-Ahead Log (WAL), eliminating disk I/O overhead. MTProto sessions update dozens of times per minute (salt rotations, sequence number increments). Performance gain is significant.

**Crash recovery (ADR-18):** A LOGGED backup table is snapshotted every 5 minutes. On crash, sessions restore from backup. Maximum data loss: 5 minutes. Users see brief reconnection, not full re-auth.

### MTProto Session Backup (LOGGED)

```sql
CREATE TABLE mtproto_sessions_backup (
    LIKE mtproto_sessions INCLUDING ALL
);

-- Index for efficient restore lookup
CREATE INDEX idx_sessions_backup_conn ON mtproto_sessions_backup(connection_id);
```

**Snapshot mechanism:** Every 5 minutes, the Go middleware truncates the backup table and copies all rows from the UNLOGGED primary. On startup, if `mtproto_sessions` is empty but `mtproto_sessions_backup` has data, restore from backup.

### Universal Connection Registry

```sql
CREATE TABLE connections (
    id              BIGSERIAL PRIMARY KEY,
    platform        TEXT NOT NULL,          -- 'telegram', 'whatsapp', etc.
    connection_type TEXT NOT NULL,          -- 'mtproto_user', 'bot_api', 'whatsapp_business'
    sf_org_id       TEXT NOT NULL,          -- Salesforce Org ID
    sf_user_id      TEXT NOT NULL,          -- Salesforce User ID
    sf_channel_id   TEXT,                  -- SF record ID (Messenger_Channel__c)
    encrypted_credentials BYTEA NOT NULL,   -- Encrypted: phone, api_id, bot_token, etc.
    proxy_url       TEXT,
    status          TEXT DEFAULT 'active',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Inbound Message Queue (Go -> Salesforce)

```sql
CREATE TABLE inbound_queue (
    id              BIGSERIAL PRIMARY KEY,
    connection_id   BIGINT REFERENCES connections(id),
    platform        TEXT NOT NULL,
    external_msg_id TEXT NOT NULL,
    chat_id         TEXT NOT NULL,
    payload         JSONB NOT NULL,
    status          TEXT DEFAULT 'pending',  -- pending | processing | delivered | failed
    retry_count     INT DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    next_retry_at   TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ
);
```

### Outbound Message Queue (Salesforce -> Telegram)

```sql
CREATE TABLE outbound_queue (
    id              BIGSERIAL PRIMARY KEY,
    connection_id   BIGINT REFERENCES connections(id),
    platform        TEXT NOT NULL,
    chat_id         TEXT NOT NULL,
    message_text    TEXT,
    media_file_id   TEXT,
    sf_message_id   TEXT,                    -- SF Messenger_Message__c record ID
    status          TEXT DEFAULT 'pending',   -- pending | processing | delivered | failed
    error_code      TEXT,                    -- Terminal error from Telegram
    error_detail    TEXT,
    retry_count     INT DEFAULT 0,
    max_retries     INT DEFAULT 5,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    next_retry_at   TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ
);
```

### Media Storage

> **Note (ADR-20):** Media metadata and files are stored in Salesforce ContentVersion, not PostgreSQL. The `media_files` table has been **eliminated** in migration 005. Platform file IDs (e.g., Telegram `file_id`) are tracked in ContentVersion custom fields. Files are linked to Message/Attachment records via ContentDocumentLink. The `credentials` column was also dropped from the `connections` table in migration 005, replaced by `encrypted_credentials` (BYTEA).

## Indexes

```sql
CREATE INDEX idx_inbound_queue_pending ON inbound_queue(status, next_retry_at)
    WHERE status IN ('pending', 'failed');

CREATE INDEX idx_outbound_queue_pending ON outbound_queue(status, next_retry_at)
    WHERE status IN ('pending', 'failed');
```

## Queue Processing with SKIP LOCKED

```sql
-- Worker picks up pending messages without blocking other workers
SELECT id, payload FROM inbound_queue
WHERE status = 'pending' AND (next_retry_at IS NULL OR next_retry_at <= NOW())
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

`SKIP LOCKED` allows multiple Go worker goroutines to process the queue concurrently without deadlocks — each worker gets different rows.

## When to Add Redis

Start with PostgreSQL only. Add Redis when:
- Horizontal scaling requires distributed rate limiting across multiple Go instances
- Real-time pub/sub between Go instances is needed (Redis Pub/Sub)
- Session read latency becomes a bottleneck (sub-millisecond reads needed at scale)
