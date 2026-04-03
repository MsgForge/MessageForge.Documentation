# Security Checklist

> Last audited: 2026-03-07 | Updated 2026-04-03 (post-ADR-20/21)

## Transport Security

- [ ] All Go server endpoints behind TLS-terminating reverse proxy (Caddy/nginx/LB)
- [ ] SSL Labs scan (PDF/screenshot) proving TLS 1.2+ enforcement
- [ ] No outdated cipher suites
- [x] Go server documented as requiring reverse proxy for TLS (`server.go`)

## Penetration Testing

- [ ] DAST scan reports (OWASP ZAP, Burp Suite, or Chimera)
- [ ] Zero high-severity vulnerabilities (XSS, SQL injection, OS command injection)
- [ ] False positives documented in supplementary report

## Authentication — Go -> Salesforce

- [x] HMAC-SHA256 on all Go -> Salesforce webhooks
- [x] Signature in `X-Signature` HTTP header
- [x] Constant-time comparison via `hmac.Equal()` in Go middleware
- [ ] Shared secrets in Salesforce Protected Custom Metadata Types

## Authentication — Salesforce -> Go

- [x] OAuth 2.0 JWT Bearer Flow for server-to-server (`salesforce/auth.go`)
- [x] Token refresh handled automatically (double-check locking, 90-min TTL)
- [ ] X.509 certificate in Connected App

## Request Validation

- [x] HMAC middleware body size limit: 10 MB (`middleware.go`)
- [x] Bot API handler body size limit: 1 MB (`handlers.go`)
- [x] Constant-time HMAC comparison (`hmac.Equal`)
- ~~[x] CF Worker timing-safe comparison (no length leak)~~ — HISTORICAL (ADR-20: Cloudflare Workers removed)

## Credential Storage

- [x] No CRM credentials hardcoded in Go source
- [x] All PostgreSQL credentials via environment variables
- [x] Bot tokens encrypted at rest in PostgreSQL (AES-256-GCM via `database/crypto.go`)
- [x] MTProto session keys in PostgreSQL UNLOGGED tables, never transmitted to Salesforce
- [ ] Salesforce secrets in Protected Custom Metadata Types (not Custom Settings)

## Memory Safety

- [x] sync.Pool buffers zeroed after use (`ingestion.go`) — prevents credential leakage
- [x] No sensitive data in error messages returned to clients

## Media Security (ADR-20: ContentVersion)

- [ ] ContentVersion file sharing rules configured (OWD, sharing sets)
- [ ] ContentDocumentLink visibility set appropriately per record
- [ ] Media files accessible only via authenticated `/sfc/servlet.shepherd/` URLs
- [ ] File size limits enforced on upload (Apex/Go validation)
- [ ] MIME type validation on upload (prevent executable uploads)

## CORS Security

- [ ] CSP Trusted Sites configured for Go middleware domain (Apex callouts)

## Data Pipeline Security

- [x] Platform Event batching implemented (5-sec flush interval, 950 KB threshold)
- [x] sync.Pool used for batch serialization buffers (prevents GC thrashing)
- [x] `Message_Delivery_Status__e` Platform Event published on terminal outbound failures
- [x] Exponential backoff for all external API retries

## Telegram-Specific

- [ ] Residential proxy pool for MTProto IP isolation
- [ ] No automated `my.telegram.org` API ID registration
- [x] Single corporate API ID used (config-driven)
- [x] FloodWait handling via token bucket + backoff (`ratelimit.go`)
- [x] Rate limiting middleware active for both MTProto and Bot API

## MTProto 2FA

- [x] SRP v6a flow implemented (`mtproto/client.go`)
- [x] Auth state machine: OTP → 2FA → success
- [x] Password never transmitted in plaintext

## Token TTL Reference

| Token | TTL | System | Notes |
|---|---|---|---|
| Salesforce OAuth | 90 min | Go → Salesforce REST API | Auto-refresh with double-check locking (`auth.go`) |

## Audit Trail

| Date | Finding | Status |
|------|---------|--------|
| 2026-03-07 | HMAC middleware missing body size limit | FIXED |
| 2026-03-07 | sync.Pool buffers not zeroed | FIXED |
| 2026-03-07 | ~~CF Worker CORS open to all origins~~ | ~~FIXED~~ — HISTORICAL (ADR-20: Cloudflare Workers removed) |
| 2026-03-07 | ~~CF Worker timingSafeEqual length leak~~ | ~~FIXED~~ — HISTORICAL (ADR-20: Cloudflare Workers removed) |
| 2026-03-07 | Go server no TLS | DOCUMENTED (reverse proxy required) |
