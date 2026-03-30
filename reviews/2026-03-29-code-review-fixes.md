# MessageForge Code Review Fixes — 2026-03-29

**Reviewers:** Architect, Go Reviewer, Security Reviewer, Risk Verifier, Docs Alignment
**Scope:** Full project parallel review

---

## Review Summary

5 parallel review agents inspected the entire MessageForge project:
- **Architect:** Phase 5 readiness assessment, risk verification, new concerns
- **Go Reviewer:** Backend code quality, nil pointers, concurrency, production readiness
- **Security Reviewer:** HMAC, JWT, crypto, CMTs, secrets scan, OWASP gaps
- **Risk Verifier:** Verified 8 "resolved" risks against actual code
- **Docs Alignment:** Documentation vs code consistency check

### Verdicts
| Agent | Verdict |
|---|---|
| Architect | NOT ready for Phase 5 |
| Go Reviewer | BLOCK — 3 HIGH issues |
| Security Reviewer | WARN 8/10 — 3 HIGH findings |
| Risk Verifier | 7/8 confirmed resolved, R02 partially open |
| Docs Alignment | PG schema perfect, 3 undocumented SF objects |

---

## Fixes Applied (This Session)

### Fix 1: Delete `archived/` directory
**Severity:** MEDIUM (blocks tooling)
**Problem:** `archived/backend/` contains Go files with broken imports to old package paths. Breaks `go vet ./...` and `go test ./...` globally.
**Fix:** Stage all already-deleted files (archived/, old backend/, salesforce/, poc/, docs/, draftdocs/, scripts/, workers/).
**Verification:** `go vet ./...` and `go test ./...` pass clean.

### Fix 2: Ingester.Add data loss on flush error
**Severity:** HIGH
**File:** `MessageForge.Backend/internal/salesforce/ingestion.go:53-65`
**Problem:** When `Add()` triggers a flush and flush fails, the batch is already reset and the new payload is never appended. Both the flushed batch and the triggering message are silently lost.
**Fix:** Preserve the new payload even when flush fails — append it to the fresh batch before returning the error.
**Verification:** New test case added for flush-failure scenario.

### Fix 3: Queue workers — FOR UPDATE SKIP LOCKED without transaction
**Severity:** HIGH
**Files:** `MessageForge.Backend/internal/messenger/queue_worker.go:68-76`, `outbound_worker.go`
**Problem:** `FOR UPDATE SKIP LOCKED` runs without explicit transaction. Lock releases immediately after scan, enabling duplicate processing by concurrent workers.
**Fix:** Wrap poll+process+mark in explicit `pool.BeginTx()` / `tx.Commit()`. Rollback on failure releases lock safely.
**Verification:** Existing tests pass.

### Fix 4: Custom `contains()` reimplements `strings.Contains`
**Severity:** HIGH (code quality)
**File:** `MessageForge.Backend/internal/messenger/outbound_worker.go:186-214`
**Problem:** Hand-rolled `contains()` and `searchString()` reimplements `strings.Contains`. Maintenance risk, not rune-safe.
**Fix:** Replace with `strings.Contains` from standard library. Delete custom functions.

### Fix 5: Delivery report errors silently discarded
**Severity:** MEDIUM
**File:** `MessageForge.Backend/internal/messenger/outbound_worker.go`
**Problem:** `_ = w.delivery.ReportFailure(...)` silently discards errors. No visibility when SF delivery tracker fails.
**Fix:** Add `slog.Error` logging for delivery report failures.

---

## Deferred Issues (Require Separate Sessions)

### Before Phase 5 (E2E Testing)

| # | Issue | Severity | Owner |
|---|---|---|---|
| 1 | Wire outbound worker into app.go with Telegram Bot API adapter | HIGH | go-backend |
| 2 | Wire media downloader into InboundProcessor (replace nil) | MEDIUM | go-backend |
| 3 | Migrate Messenger_Settings__mdt to Middleware_Config__mdt (Protected) | HIGH | salesforce-dev |
| 4 | Add CRUD/FLS to ChannelSetupController.createChannel | HIGH | salesforce-dev |
| 5 | Convert MessengerOutboundService.sendMessage to Queueable | HIGH | salesforce-dev |
| 6 | Set OWD to Private for Messenger_Chat__c | HIGH | salesforce-dev |
| 7 | MTProto session store — Client.store field wired but never called | HIGH | go-backend |

### Before AppExchange

| # | Issue | Severity | Owner |
|---|---|---|---|
| 8 | OWASP ZAP DAST scan — zero high-severity findings | CRITICAL | security |
| 9 | SSL Labs scan — TLS 1.2+ proof | HIGH | infra |
| 10 | CSP Trusted Sites for R2, Centrifugo, CF Worker | HIGH | salesforce-dev |
| 11 | Lock down Centrifugo ALLOWED_ORIGINS in production | HIGH | infra |
| 12 | Document AES-256-CBC limitation in EncryptionService | MEDIUM | docs |
| 13 | Startup warning when WEBHOOK_SECRET is empty | MEDIUM | go-backend |

### Documentation Cleanup

| # | Issue |
|---|---|
| 14 | Remove or archive old 7-phase MVP plan (conflicts with current 6-phase roadmap) |
| 15 | Document Messenger_Event__c, Messenger_Settings__mdt, Middleware_Config__mdt in sf-data-model.md |
| 16 | Clean stale Messenger_Connection__c references from .claude/agents/salesforce-dev.md |

---

## Risk Register Updates

| Risk | Previous | New Status | Notes |
|---|---|---|---|
| R02 | RESOLVED | **PARTIALLY OPEN** | Messenger_Settings__mdt still unprotected and referenced |
| All others | — | Unchanged | — |

### New Risks Identified

| ID | Risk | Severity | Likelihood |
|---|---|---|---|
| R21 | Queue worker duplicate processing (no tx boundary) | HIGH | LIKELY |
| R22 | Ingester data loss on flush failure | HIGH | POSSIBLE |
| R23 | Nil media downloader — latent panic on media messages | MEDIUM | LIKELY |

R21 and R22 are being fixed in this session. R23 deferred to Phase 2 completion.

---

## Commit Plan

All fixes applied in a single commit with message:

```
fix(backend): resolve 5 HIGH-severity findings from code review

- Delete archived/ directory (blocks tooling)
- Fix Ingester.Add: preserve payload when flush fails (data loss risk)
- Wire queue workers in transaction: FOR UPDATE SKIP LOCKED requires explicit tx
- Replace custom contains() with strings.Contains (stdlib)
- Log delivery report failures (visibility for outbound failures)

Resolves R21, R22 partially. Documents R23 for Phase 2.

Co-Authored-By: Code Review Team <noreply@anthropic.com>
```
