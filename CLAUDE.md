# MessageForge Documentation

Central documentation repository for the MessageForge platform — a multi-messenger to Salesforce CRM integration suite.

## Developer Context

Primary developer is a Go expert with limited Salesforce experience.

## Repository Map

```
architecture/
├── architecture.md              — System overview, data flows, component roles
└── adr.md                       — 21 Architectural Decision Records with rationale

reference/
├── postgres-schema.md           — PostgreSQL tables, indexes, queue patterns
├── sf-data-model.md             — Salesforce custom objects (21 + 2 CMTs), Platform Events, package structure
├── salesforce-guide.md          — Salesforce concepts explained for Go developers
├── governor-limits.md           — SF governor limits and design rules
├── security-checklist.md        — Security audit checklist (Go, Apex, LWC, infra)
├── sf-messenger-erd-v9-svg.html — ERD diagram (visual)
└── style_guide_cursor_rules.md  — Go architecture style guide (squad pattern)

plans/
├── mvp-implementation-plan.md   — 7-phase, ~30 task implementation plan
├── appexchange-onboarding.md    — AppExchange security review & publishing process
├── future-multi-messenger.md    — Adapter pattern for WhatsApp, Messenger, Viber
├── media-storage-migration-plan.md — (DEPRECATED) R2→ContentVersion migration
├── 2026-03-07-go-rewrite-design.md — Backend rewrite design (historical, predates ADR-20/21)
├── 2026-03-07-go-rewrite-plan.md   — Backend rewrite task plan (historical, predates ADR-20/21)
├── 2026-03-09-telegram-poc-design.md — PoC design document
└── 2026-03-09-telegram-poc-plan.md   — PoC implementation plan

reviews/
├── 2026-03-29-architecture-review.md  — Full architectural feasibility assessment
├── 2026-03-29-risk-register.md        — 20 risks with severity, mitigation, agent commands
└── 2026-03-29-implementation-roadmap.md — 6-phase agent-executable implementation plan

research/
├── complete-technical-specification.md — Full architecture & technical reference (single doc)
├── Salesforce-Telegram Integration Technical Plan.md — Deep research: protocols, auth, limits
├── Salesforce-Telegram Integration Refinements.md   — Edge cases: JWT, SRP, media, batching
├── system_arch_research.md      — Initial system architecture research
└── archive/                     — Historical research (PROMPT files, salesforce-files analysis)
```

## Architecture Overview

```
Telegram <-> Go Middleware <-> Salesforce
                |                  |
         PostgreSQL          Platform Events (empApi) -> LWC
         (sessions, queues)        |
                              ContentVersion (media files)
```

## Key Decisions (Summary)

| ADR | Decision |
|---|---|
| 1 | Both MTProto and Bot API supported bidirectionally |
| 2 | Remote PostgreSQL as Go-side data store |
| 3 | ~~Cloudflare R2 + Workers for media~~ **SUPERSEDED by ADR-20** |
| 4 | ~~Centrifugo for real-time LWC chat~~ **SUPERSEDED by ADR-21** |
| 5 | Single corporate API ID + residential proxy pool |
| 6 | HMAC-SHA256 for webhook authentication |
| 7 | Hybrid ingestion: Platform Events + Bulk API 2.0 + REST |
| 8 | Pub/Sub API (gRPC) for Go <-> Salesforce event streaming |
| 9 | Hetzner (Europe) primary hosting |
| 10 | Messenger-agnostic Go interface design (adapter pattern) |
| 11 | 2GP Managed Package |
| 12 | Generic `Messenger_*` naming (not `Telegram_*`) |
| 13 | ~~Centrifugo JWT auth via Apex~~ **SUPERSEDED by ADR-21** |
| 14 | ~~Direct-to-R2 outbound media via SigV4 presigned URLs~~ **SUPERSEDED by ADR-20** |
| 15 | SRP v6a for MTProto 2FA |
| 16 | Platform Events for outbound delivery failure sync |
| 17 | sync.Pool + streaming JSON encoder for batch byte tracking |
| 18 | UNLOGGED tables + periodic LOGGED backup for session durability |
| 19 | Data Flow Routing Matrix — clear API assignment per data flow |
| 20 | Salesforce ContentVersion for all media (supersedes ADR-3, ADR-14) |
| 21 | Platform Events (empApi) for real-time UI (supersedes ADR-4, ADR-13) |

Full rationale: `architecture/adr.md`

## Related Repos

| Repo | Purpose |
|---|---|
| **MessageForge.Backend** | Go middleware (MTProto, Bot API, PostgreSQL, Platform Events publisher) |
| **MessageForge.Salesforce** | Salesforce managed package (Apex, LWC, 2GP, namespace: tgint__) |
| **MessageForge.PoC** | Telegram proof-of-concept demo |
| **MessageForge.Admin** | Administration dashboard |
