# Full Verification Prompt — MessageForge Code vs Documentation

**Date:** 2026-04-03
**Purpose:** Double-check that all code in MessageForge.Backend and MessageForge.Salesforce matches the architecture, ADRs, plans, and security requirements documented in MessageForge.Documentation.

---

## Prompt (copy everything below into a new Claude Code session)

```
You are a senior auditor performing a comprehensive verification of the MessageForge project.
Your job is to read ALL documentation and compare it against the ACTUAL code.
Flag every discrepancy — stale docs, unimplemented features, violated ADRs, missing security controls.

Use /evaluation skill to score each area. Use question-resolver agent for ambiguous decisions.
Do NOT ask the user — only notify at completion with a structured report.

## Repositories

- Go backend: /Users/levshabalin/Repositories/SF/MessageForge.Backend
- Salesforce package: /Users/levshabalin/Repositories/SF/MessageForge.Salesforce
- Documentation: /Users/levshabalin/Repositories/SF/MessageForge.Documentation

## Phase 1: Read All Documentation (do NOT skip any file)

Read these files IN FULL before touching any code:

### Architecture & ADRs
1. MessageForge.Documentation/architecture/architecture.md
2. MessageForge.Documentation/architecture/adr.md (ALL ADRs — check every single one)

### Plans
3. MessageForge.Documentation/plans/mvp-implementation-plan.md
4. MessageForge.Documentation/plans/media-storage-migration-plan.md
5. MessageForge.Documentation/plans/appexchange-onboarding.md
6. MessageForge.Documentation/plans/future-multi-messenger.md

### Reference
7. MessageForge.Documentation/reference/postgres-schema.md
8. MessageForge.Documentation/reference/sf-data-model.md
9. MessageForge.Documentation/reference/security-checklist.md
10. MessageForge.Documentation/reference/deployment-guide.md
11. MessageForge.Documentation/reference/governor-limits.md
12. MessageForge.Documentation/reference/salesforce-guide.md

### Prior Reviews
13. MessageForge.Documentation/reviews/2026-03-29-architecture-review.md
14. MessageForge.Documentation/reviews/2026-03-29-risk-register.md
15. MessageForge.Documentation/reviews/2026-03-30-fix-register.md
16. MessageForge.Documentation/reviews/2026-03-30-recheck-review.md
17. MessageForge.Documentation/reviews/security-findings.md
18. MessageForge.Documentation/reviews/cost-model.md

## Phase 2: Verify Code Against Documentation

For EACH area below, launch parallel agents where possible.

### 2.1 ADR Compliance Audit

Read every ADR from adr.md. For each ADR with status "accepted":
- Find the corresponding code implementation
- Verify the code follows the decision (not an earlier/rejected approach)
- Flag ADRs that reference deleted/moved code (e.g., R2, Centrifugo after ADR-20/21)
- Flag code that contradicts an accepted ADR

Pay special attention to:
- **ADR-20 (Media in SF ContentVersion, not R2)** — verify NO R2 upload code is active
- **ADR-21 (Remove Centrifugo)** — verify Centrifugo code status (dead code? still wired?)
- **ADR-18 (Session backup)** — verify UNLOGGED table + backup mechanism
- **ADR-14 (HMAC auth)** — verify both Go and Apex sides use HMAC-SHA256

### 2.2 Schema Verification

Compare `MessageForge.Documentation/reference/postgres-schema.md` against:
- `MessageForge.Backend/internal/database/migrations/*.sql` (actual schema)
- `MessageForge.Backend/internal/messenger/outbound_store.go` (column names in queries)
- `MessageForge.Backend/internal/messenger/queue_worker.go` (column names in queries)
- `MessageForge.Backend/internal/database/session_backup.go` (table names)

Flag: column name mismatches, missing tables, documented-but-not-created tables, created-but-not-documented tables.

### 2.3 SF Data Model Verification

Compare `MessageForge.Documentation/reference/sf-data-model.md` against:
- `MessageForge.Salesforce/force-app/main/default/objects/` (actual custom objects)
- `MessageForge.Salesforce/force-app/main/default/classes/InboundMessageTriggerHandler.cls` (field references)
- `MessageForge.Backend/internal/salesforce/ingestion.go` (Platform Event field names)
- `MessageForge.Backend/internal/salesforce/delivery_status.go` (Platform Event field names)

Flag: field name mismatches between Go constants and SF object definitions, missing fields, `tgint__` namespace violations.

### 2.4 Security Checklist Audit

Read `MessageForge.Documentation/reference/security-checklist.md` and verify EACH item:

**Go backend:**
- HMAC webhook validation (server/middleware.go or similar)
- No hardcoded secrets (grep for API keys, tokens, passwords in source)
- Parameterized SQL queries only (no string concatenation in SQL)
- Input validation on all API endpoints
- Rate limiting on webhook endpoints
- Encryption at rest for credentials (migration 003/004)
- TLS enforcement on outbound HTTP calls

**Salesforce:**
- `with sharing` on all service classes
- FLS/CRUD checks (Security.stripInaccessible or WITH SECURITY_ENFORCED)
- HMAC validation on inbound REST endpoints
- No hardcoded secrets in Apex (use Custom Metadata)
- SOQL injection prevention (no dynamic SOQL with user input)
- HTTPS enforcement on callouts

Run security-reviewer agent on both repos.

### 2.5 MVP Plan Status

Read `MessageForge.Documentation/plans/mvp-implementation-plan.md`.
For each phase/task listed:
- Check if implemented in code
- Mark as: DONE / PARTIAL / NOT STARTED / CONTRADICTED
- Flag any task marked as "done" in docs but missing in code

### 2.6 API Contract Verification

Verify the Go↔SF API contracts match:
- Go's `POST /api/outbound` handler expects fields X — does SF's `MessengerOutboundService.cls` send exactly those fields?
- Go's ingester publishes Platform Event with fields Y — does SF's `InboundMessageTriggerHandler.cls` read exactly those fields?
- Go's delivery tracker publishes fields Z — does SF's `DeliveryStatusTriggerHandler.cls` read exactly those fields?
- HMAC secret name/format matches between Go config and SF Custom Metadata

### 2.7 Dead Code Audit

Identify code that should have been removed per ADRs but wasn't:
- R2/Cloudflare references (ADR-20 says use ContentVersion)
- Centrifugo wiring if ADR-21 says remove it
- MTProto references if MVP is Bot API only
- Unused imports, unreachable functions, commented-out blocks

Run: `go vet ./...` and check for unused variables/imports.
Search for: TODO, FIXME, HACK, XXX comments.

### 2.8 Test Coverage Check

Run: `cd MessageForge.Backend && go test -cover ./...`
For each package:
- Report coverage percentage
- Flag packages below 80%
- Flag critical paths with no tests (auth, ingestion, outbound worker, HMAC validation)

Run Salesforce tests: report any test classes and their status.

### 2.9 Deployment Guide Verification

Read `MessageForge.Documentation/reference/deployment-guide.md` and verify:
- All env vars listed actually exist in config.go
- All env vars in config.go are documented in the guide
- Docker-compose services match what the guide describes
- Any manual steps documented are still necessary

## Phase 3: Cross-Reference Prior Reviews

Read the risk register (2026-03-29) and fix register (2026-03-30).
For each risk/fix item:
- Is it resolved in current code?
- Is it still relevant?
- Flag items marked "fixed" that are actually still broken

## Phase 4: Generate Report

Output a structured report:

### Format

# MessageForge Verification Report — 2026-04-03

## Executive Summary
[2-3 sentences: overall health, critical issues count, deployment readiness score 0-100]

## ADR Compliance
| ADR | Title | Status | Code Match | Issues |
|-----|-------|--------|------------|--------|

## Schema Alignment
| Source | Documented | Actual | Match? | Issue |
|--------|-----------|--------|--------|-------|

## Security Findings
| Severity | Area | Finding | File:Line | Recommendation |
|----------|------|---------|-----------|----------------|

## MVP Progress
| Phase | Task | Status | Evidence |
|-------|------|--------|----------|

## API Contract Mismatches
| Endpoint | Go Expects | SF Sends | Match? |
|----------|-----------|----------|--------|

## Dead Code
| File | Type | Reason | ADR Reference |
|------|------|--------|---------------|

## Test Coverage
| Package | Coverage | Critical? | Recommendation |
|---------|----------|-----------|----------------|

## Stale Documentation
| Document | Section | Issue | Recommendation |
|----------|---------|-------|----------------|

## Risk Register Status
| Risk ID | Original Finding | Current Status | Evidence |
|---------|-----------------|----------------|----------|

## Action Items (Priority Order)
1. [CRITICAL] ...
2. [HIGH] ...
3. [MEDIUM] ...
4. [LOW] ...

---

Write the final report to:
MessageForge.Documentation/reviews/2026-04-03-full-verification-report.md

## Rules

- Do NOT modify any code. This is READ-ONLY verification.
- Do NOT ask the user questions. Use question-resolver agent.
- Do NOT skip any file listed above. Read every one.
- Use parallel agents for independent checks (schema, security, tests, dead code).
- Be precise: include file paths and line numbers for every finding.
- Distinguish between "doc is wrong" vs "code is wrong" — never assume docs are always right.
- If a document references a file that doesn't exist, flag it.
- If code implements something not documented, flag it.
```
