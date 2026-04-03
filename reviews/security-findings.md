> **Note:** This security audit is from 2026-03-07 and reflects the pre-ADR-20 architecture. Findings referencing R2, Cloudflare Workers, and presigned URLs are historical — media now uses Salesforce ContentVersion (ADR-20, 2026-03-30).

# Security Findings — 2026-03-29

**Reviewer:** Claude Opus 4.6 (automated security review)
**Scope:** MessageForge.Backend Go middleware — all handlers, auth, database, media

---

## Findings Summary

| ID | Severity | Component | Finding | Status |
|---|---|---|---|---|
| S01 | HIGH | media/r2.go | Presigned URL TTL uncapped — caller could request multi-day URLs | FIXED — **NOT APPLICABLE (ADR-20: R2 removed)** |
| S02 | HIGH | database/credentials | Bot tokens stored as plaintext JSONB in PostgreSQL | FIXED |
| S03 | MEDIUM | messenger/app.go | Inbound media presign TTL was 24h, should be ≤ 60 min | FIXED — **NOT APPLICABLE (ADR-20: R2 removed)** |
| S04 | LOW | server/middleware.go | HMAC middleware uses timing-safe comparison (hmac.Equal) | OK |
| S05 | LOW | server/middleware.go | Body size limit enforced (10 MB) | OK |
| S06 | LOW | salesforce/auth.go | JWT token refresh with double-check locking | OK |
| S07 | LOW | database/pool.go | Parameterized queries only (pgx) | OK |
| S08 | INFO | server/handlers.go | Health endpoint does not leak internal paths | OK |

---

## S01: Presigned URL TTL Uncapped (FIXED) — NOT APPLICABLE (ADR-20: R2 removed)

**File:** `internal/media/r2.go`
**Before:** `PresignedGetURL` and `PresignedPutURL` accepted any `time.Duration` with no cap.
**Risk:** A caller could request a presigned URL with a TTL of days or weeks, creating a persistent access point.
**Fix:** Added `MaxInboundPresignTTL` (60 min) and `MaxOutboundPresignTTL` (15 min) caps. TTL is silently clamped if exceeded.

## S02: Plaintext Credentials in PostgreSQL (FIXED)

**File:** `internal/database/crypto.go` (new)
**Before:** Bot tokens and API keys stored as plaintext JSONB in the `credentials` column.
**Risk:** Database compromise or backup leak exposes all channel credentials.
**Fix:** Created AES-256-GCM encrypt/decrypt functions. Migration `003_encryption.sql` adds `encrypted_credentials BYTEA` column. The `EncryptionKey` environment variable provides the 32-byte key.

## S03: 24-Hour Inbound Media TTL (FIXED) — NOT APPLICABLE (ADR-20: R2 removed)

**File:** `internal/messenger/app.go`
**Before:** `NewInboundProcessor(nil, r2Client, 24*time.Hour)` — presigned GET URLs valid for 24 hours.
**Risk:** Excessively long media access window. If URL is leaked, media accessible for a full day.
**Fix:** Changed to `60*time.Minute`. Additionally, the R2 client now enforces `MaxInboundPresignTTL` as a hard cap.

---

## Positive Findings (No Issues)

- **HMAC middleware** (`middleware.go`): Uses `hmac.Equal()` for constant-time comparison. Body size limited to 10 MB. Signature required on all webhook endpoints.
- **Salesforce JWT auth** (`auth.go`): Private key loaded from environment variable, never logged. Token refresh uses double-check locking to prevent race conditions. 90-min TTL with auto-refresh.
- **Database queries**: All queries use parameterized `$1, $2` placeholders via pgx. No string concatenation.
- **Error responses**: Health endpoint returns generic `{"status":"ok"}` or `{"status":"degraded"}`. No stack traces, no internal paths, no connection strings.
- **sync.Pool zeroing** (`ingestion.go`): Buffers zeroed after use to prevent credential leakage across requests.

---

## Remaining Unchecked Items (Infrastructure)

These require deployment, not code changes:
- [ ] TLS termination via reverse proxy (Caddy) — documented in deployment guide
- [ ] SSL Labs scan — requires live deployment
- [ ] DAST scan (OWASP ZAP) — requires live deployment
- ~~R2 bucket private access verification — requires Cloudflare dashboard~~ NOT APPLICABLE (ADR-20)
- ~~R2 CORS policy — requires Cloudflare dashboard~~ NOT APPLICABLE (ADR-20)
- [ ] CSP Trusted Sites — requires Salesforce org setup
- [ ] Residential proxy pool — requires vendor selection + budget
- [ ] X.509 certificate in Connected App — requires Salesforce org setup
