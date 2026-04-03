# Documentation Cleanup & Consolidation — Full Audit

## Objective

Audit and update ALL project documentation to reflect the current architecture after three major decisions:

1. **ADR-20 (2026-03-30): R2 eliminated → ContentVersion for all media** — Cloudflare R2, Workers, SigV4 presigned URLs, HMAC media URLs — all removed. Media stored in Salesforce ContentVersion via REST API. Supersedes ADR-3 and ADR-14.
2. **ADR-21 (2026-04-01): Centrifugo eliminated → Platform Events (empApi) for real-time** — External WebSocket server removed. LWC uses `lightning/empApi` to subscribe to Platform Events. Auth flow uses polling via Apex callout. Supersedes ADR-4 and ADR-13.
3. **Any other stale references** — scan ALL docs for contradictions with current ADRs.

**This is NOT a rewrite.** This is a surgical cleanup: find stale references, update them to match current ADRs, and consolidate scattered docs.

---

## Architecture Context (Current State After ADR-20 + ADR-21)

```
Telegram <-> Go Middleware <-> Salesforce
                |                  |
         PostgreSQL          Platform Events (empApi) -> LWC
         (sessions, queues)        |
                              ContentVersion (media files)
```

**Eliminated components:** Centrifugo, Cloudflare R2, Cloudflare Workers, CSP Trusted Sites for WebSocket, CentrifugoTokenController, SigV4 presigned URLs, HMAC media auth, `realtime/` Go package, `media_files` PostgreSQL table.

**Replacement mapping:**
| Old | New | ADR |
|---|---|---|
| Centrifugo WebSocket | Platform Events via `lightning/empApi` in LWC | ADR-21 |
| Centrifugo JWT auth from Apex | Removed (no external auth needed) | ADR-21 |
| WebSocket for Telegram auth flow | Polling via Apex callout to Go middleware | ADR-21 |
| Cloudflare R2 storage | Salesforce ContentVersion (Files) | ADR-20 |
| Cloudflare Workers CDN | `/sfc/servlet.shepherd/` URLs (native SF) | ADR-20 |
| SigV4 presigned upload URLs | ContentVersion REST API (multipart) | ADR-20 |
| HMAC media display auth | Salesforce session-based access | ADR-20 |
| `media_files` PG table | ContentDocumentLink references | ADR-20 |
| `.env` Centrifugo vars | Removed | ADR-21 |
| `.env` R2 vars | Removed | ADR-20 |

---

## Execution Strategy

### Phase 1: AUDIT — Find all stale references (parallel agents)

Launch 4 agents in parallel, each scanning a different scope:

**Agent 1 — Core docs audit:**
Scan these files for ANY reference to Centrifugo, R2, Cloudflare, Workers, WebSocket (external), presigned URL, SigV4, HMAC media, `media_files` table:
- `MessageForge.Documentation/architecture/architecture.md`
- `MessageForge.Documentation/architecture/adr.md`
- `MessageForge.Documentation/reference/postgres-schema.md`
- `MessageForge.Documentation/reference/sf-data-model.md`
- `MessageForge.Documentation/reference/security-checklist.md`
- `MessageForge.Documentation/reference/deployment-guide.md`
- `MessageForge.Documentation/reference/governor-limits.md`
- `MessageForge.Documentation/reference/salesforce-guide.md`
- `MessageForge.Documentation/CLAUDE.md`
- `CLAUDE.md`

Output: file:line — what's stale — what it should say.

**Agent 2 — Plans & roadmaps audit:**
Scan these files:
- `MessageForge.Documentation/plans/mvp-implementation-plan.md`
- `MessageForge.Documentation/plans/appexchange-onboarding.md`
- `MessageForge.Documentation/plans/future-multi-messenger.md`
- `MessageForge.Documentation/plans/media-storage-migration-plan.md`
- `MessageForge.Documentation/reviews/2026-03-29-implementation-roadmap.md`
- `MessageForge.Documentation/reviews/2026-03-29-risk-register.md`
- `MessageForge.Documentation/reviews/2026-03-29-architecture-review.md`
- `MessageForge.Documentation/reviews/2026-03-30-fix-register.md`
- `MessageForge.Documentation/reviews/2026-03-30-recheck-review.md`
- `MessageForge.Documentation/reviews/cost-model.md`

Output: same format.

**Agent 3 — Skills, agents, commands audit:**
Scan these files:
- `.claude/skills/media-pipeline.md` — likely needs full rewrite for ContentVersion
- `.claude/skills/data-ingestion.md`
- `.claude/skills/salesforce-apex-patterns.md`
- `.claude/skills/telegram-auth-flow.md`
- `.claude/skills/question-resolver.md`
- `.claude/skills/outbound-failure-sync.md`
- `.claude/agents/go-backend.md`
- `.claude/agents/salesforce-dev.md`
- `.claude/agents/security-reviewer.md`
- `.claude/agents/question-resolver.md`
- `.claude/commands/route.md`
- `docs/claude/commands.md`
- `docs/claude/conventions.md`
- `.env.example`
- `.gitignore`

Output: same format.

**Agent 4 — Web research for ContentVersion best practices:**
Use WebSearch to find:
- "Salesforce ContentVersion REST API upload example Apex"
- "Salesforce ContentVersion LWC display image"
- "Salesforce ContentVersion file size limits per edition 2025 2026"
- "Salesforce ContentVersion storage cost per GB"
- "AppExchange managed package ContentVersion best practices"
- "Salesforce empApi LWC Platform Events subscribe example"
- "Salesforce empApi best practices 2025 2026"

Output: key facts with source URLs for updating the media-pipeline skill.

### Phase 2: FIX — Apply all changes (parallel agents by scope)

Based on Phase 1 audit results, launch 3 parallel agents:

**Agent A — Update core architecture docs:**
- Merge ADR-20 into `adr.md` table (currently only in separate file `adr-20-media-storage-pivot.md`)
- Mark ADR-3, ADR-14 as SUPERSEDED in the table (like we did for ADR-4, ADR-13)
- Update `architecture.md` — remove R2/Workers from component table and data flow diagram
- Update `CLAUDE.md` — remove "Never store media in Salesforce" rule, remove R2 from Backend stack
- Update `MessageForge.Documentation/CLAUDE.md` — update architecture overview diagram
- Update `.env.example` — remove R2 env vars
- Update `.gitignore` if needed
- Update `postgres-schema.md` — remove `media_files` table reference
- Update `sf-data-model.md` — add ContentVersion patterns if not already there
- Update `deployment-guide.md` — remove Cloudflare setup sections
- Update `security-checklist.md` — remove R2/Workers security checks, add ContentVersion checks

**Agent B — Update skills and agents:**
- Rewrite `.claude/skills/media-pipeline.md` — replace R2 pipeline with ContentVersion pipeline (use Web research from Phase 1 Agent 4)
- Update all remaining R2 references in skills/agents (from Phase 1 Agent 3 findings)
- Ensure `salesforce-apex-patterns.md` package structure is current
- Update `go-backend.md` — remove R2 env vars, update directory structure

**Agent C — Update plans and reviews:**
- Add deprecation headers to stale plans/reviews that reference old architecture
- Update `mvp-implementation-plan.md` — remove R2/Centrifugo phases, add ContentVersion/empApi phases
- Update `appexchange-onboarding.md` — remove external dependency sections (R2, Centrifugo)
- Update risk register — remove R2/Centrifugo risks, add ContentVersion storage risks

### Phase 3: VERIFY — Self-evaluation loop

After all fixes applied, run a verification loop:

1. `grep -r "Centrifugo\|centrifugo" . --include="*.md" | grep -v worktrees | grep -v research/PROMPT | grep -v node_modules` — must return ZERO hits (except in ADR superseded notes and competitor research)
2. `grep -r "R2\|Cloudflare\|cloudflare\|presigned\|SigV4" . --include="*.md" | grep -v worktrees | grep -v research/ | grep -v node_modules` — must return ZERO hits (except in ADR superseded notes)
3. `grep -r "CENTRIFUGO_\|R2_\|CLOUDFLARE" . --include="*.md" | grep -v worktrees | grep -v research/` — must return ZERO env var references
4. Read `CLAUDE.md` and verify:
   - No Centrifugo in structure table
   - No R2 in structure table
   - No "Never store media in Salesforce" rule
   - Skills index is current
   - ADR summary table matches `adr.md`
5. Read `architecture.md` and verify the data flow diagram has no eliminated components

If any verification fails: fix and re-verify until clean.

### Phase 4: CONSOLIDATE — Clean up scattered docs

1. **Research archive:** Move all `research/salesforce-files-*` and `research/PROMPT-*` files into a `research/archive/` subdirectory. These are historical research, not active docs.
2. **ADR consolidation:** Ensure `adr.md` is the single source of truth for all ADR summaries. Remove `adr-20-media-storage-pivot.md` as a standalone file (content already in `adr.md` after Phase 2).
3. **Update `MessageForge.Documentation/CLAUDE.md` Repository Map** — ensure it reflects current file structure after moves.

---

## Rules

1. **Do NOT modify `research/` files** (except moving to archive). They are historical records.
2. **Do NOT modify worktree copies** (`.claude/worktrees/`). They sync separately.
3. **Preserve ADR history** — mark as SUPERSEDED, never delete ADR content.
4. **Use the architect agent** for evaluating whether architecture.md and data flow diagrams are internally consistent.
5. **Use WebSearch** when you need current SF API details (ContentVersion limits, empApi patterns) to write accurate replacement content.
6. **Every edit must be traceable** to a specific ADR decision.
7. **Self-evaluate after each phase** before proceeding to the next.

---

## Expected Outcome

After execution, a developer reading ANY documentation file will see a consistent picture:
- **Real-time:** Platform Events → empApi → LWC
- **Media:** ContentVersion REST API → `/sfc/servlet.shepherd/` display
- **No mention of:** Centrifugo, R2, Cloudflare Workers, external WebSocket, presigned URLs
- **ADR table:** Complete with all 21 ADRs, superseded ones clearly marked
