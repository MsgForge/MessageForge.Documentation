# SINGLE-RUN PROMPT: Salesforce Files Media Storage — Solution Research

> **How to use:** Copy everything below the `---` line into a NEW Claude Code session. It will autonomously launch parallel research agents, wait for results, consolidate into a unified solution document + ADR. One session, one run, fully autonomous.

---

You are an orchestrator. We have DECIDED to store all media files (photos, videos, voice, documents, stickers) in Salesforce ContentDocument/ContentVersion instead of Cloudflare R2. This is NOT up for debate — the decision is final.

**Your job: research HOW to make this work. Find solutions for every technical challenge. Produce documentation that enables implementation.**

If you find a limitation or risk — find a workaround, not a reason to say no.

## DECISION CONTEXT (NON-NEGOTIABLE)

**What:** Replace Cloudflare R2 + Workers media storage with Salesforce ContentDocument/ContentVersion (standard Salesforce Files).

**Why (business, final):**
1. Files stored in client's Salesforce org — zero external storage liability for us
2. Salesforce sells storage space — our package driving consumption = aligned AppExchange revenue
3. Competitor SMS Ninja uses this exact approach — proven on AppExchange
4. We WANT this. Period.

**Current architecture being replaced:** ADR-3 (R2 + Workers for media) and ADR-14 (SigV4 presigned URLs). Both superseded.

## PROJECT CONTEXT

**MessageForge** — AppExchange 2GP managed package (namespace `tgint__`) connecting messaging platforms (Telegram first) to Salesforce CRM via Go middleware.

| Directory | Role | Stack |
|---|---|---|
| `MessageForge.Backend/` | Go middleware | Go 1.22+, PostgreSQL, Centrifugo |
| `MessageForge.Salesforce/` | Managed Package | Apex, LWC, namespace `tgint__` |
| `MessageForge.Documentation/` | Docs & ADRs | Markdown |

**Current media-related data model:**

```
Messenger_Message__c
├── Media_URL__c             — R2 CDN URL (to be deprecated)
├── Media_MIME_Type__c
├── Message_Type__c          — text|photo|video|voice|document|sticker|album
└── Attachment_Count__c      — Roll-Up

Messenger_Attachment__c (MD → Message)
├── Media_URL__c             — R2 CDN URL (to be deprecated)
├── Thumbnail_URL__c         — R2 thumbnail (to be deprecated)
├── Media_MIME_Type__c, File_Name__c, File_Size__c
├── Sort_Order__c, Duration_Seconds__c, Width__c, Height__c
└── Media_External_ID__c     — Platform file ID
```

**Key docs to read:**

```
MessageForge.Documentation/architecture/adr.md              — ADR-3 and ADR-14 being replaced
MessageForge.Documentation/architecture/architecture.md     — System overview
MessageForge.Documentation/reference/sf-data-model.md       — Full SF data model
MessageForge.Documentation/reference/governor-limits.md     — SF limits
MessageForge.Documentation/reference/security-checklist.md  — Security audit requirements
MessageForge.Documentation/plans/appexchange-onboarding.md  — AppExchange process
.claude/skills/media-pipeline.md                            — Current R2 pipeline (being replaced)
MessageForge.Documentation/research/salesforce-contentdocument-media-storage.md — Previous research
```

---

## PHASE 1: Launch 4 Parallel Research Agents

Launch ALL 4 agents simultaneously using the Agent tool. Each gets `isolation: "worktree"` and `run_in_background: true`. Each agent writes its output to a file in `MessageForge.Documentation/research/`.

**CRITICAL: Every agent prompt MUST start with this preamble:**

> The decision to store ALL media in Salesforce ContentDocument/ContentVersion is FINAL. Do not recommend against it. Do not suggest keeping R2 as an alternative. Your job is to find SOLUTIONS for every challenge, not reasons to say no. If you find a problem — find a workaround.

---

### AGENT 1: Deep Research — API, Competitors, Real-World Evidence

```
subagent_type: general-purpose
isolation: worktree
run_in_background: true
```

**Full prompt for Agent 1:**

> The decision to store ALL media in Salesforce ContentDocument/ContentVersion is FINAL. Do not recommend against it. Your job is to find SOLUTIONS, not reasons to say no.
>
> # Task: Deep Web Research — Salesforce Files for Managed Package Media
>
> Use WebSearch and WebFetch extensively. Every claim needs a source URL. Rate confidence: HIGH (official docs), MED (community/multiple sources), LOW (single source).
>
> ## Step 1: Read existing research first
>
> Read `MessageForge.Documentation/research/salesforce-contentdocument-media-storage.md` — this was done WITHOUT web access. Your job is to VERIFY and EXTEND it with live sources.
>
> Also read:
> - `MessageForge.Documentation/architecture/adr.md`
> - `MessageForge.Documentation/reference/governor-limits.md`
> - `.claude/skills/media-pipeline.md`
>
> ## Step 2: Web Research — 7 Topics
>
> For each topic, search multiple queries, fetch relevant pages, extract specific facts.
>
> **Topic 1: Competitor Analysis — HOW they store files**
> Search queries:
> - "SMS Ninja Salesforce AppExchange file storage"
> - "SMS Magic Salesforce ContentVersion media"
> - "Mogli SMS Salesforce file attachment"
> - "Heymarket Salesforce ContentDocument"
> - "Salesforce messaging app ContentVersion AppExchange"
> - "Salesforce digital engagement file storage"
> For each app found: document whether ContentVersion or external, and how they display in UI.
>
> **Topic 2: ContentVersion REST API — Verify Everything**
> Search + fetch Salesforce developer docs:
> - "Salesforce REST API ContentVersion insert blob multipart"
> - "Salesforce ContentVersion REST API binary upload example"
> - "Salesforce FirstPublishLocationId custom object"
> - Fetch: developer.salesforce.com ContentVersion reference
> Verify: max file size (2GB?), multipart format, required fields, FirstPublishLocationId with namespaced objects
>
> **Topic 3: Storage Pricing & What Happens When Full**
> - "Salesforce file storage pricing per GB 2025 2026"
> - "Salesforce file storage limit exceeded behavior"
> - "Salesforce Enterprise edition file storage default"
> - "Salesforce additional file storage cost"
> Calculate: 100-user Enterprise org = how much file storage?
> What happens when storage quota exceeded? Block uploads? Warning? Grace period?
>
> **Topic 4: Video — Solving the Streaming Problem**
> The concern: Salesforce servlet may not support HTTP Range requests = no seeking.
> - "Salesforce sfc servlet shepherd range request video"
> - "Salesforce ContentVersion video streaming LWC"
> - "HTML5 video Salesforce Files playback"
> - "Salesforce file download servlet byte range"
> If no range requests: what's the WORKAROUND? (e.g., chunked download, progressive download, separate thumbnail + full download link)
> This is NOT a reason to say no — find the solution.
>
> **Topic 5: API Call Budget — Real Numbers**
> - "Salesforce REST API ContentVersion upload count against API limit"
> - "Salesforce API call limit Enterprise Unlimited"
> - "Salesforce connected app API limit increase"
> - "Salesforce FirstPublishLocationId API call count"
> Model: 50 / 500 / 5000 media messages per day. How many API calls? What % of daily budget?
> If heavy usage exceeds budget: what's the SOLUTION? (API add-ons, Unlimited edition, batching strategies)
>
> **Topic 6: AppExchange Security Review — ContentVersion Requirements**
> - "AppExchange security review ContentVersion requirements"
> - "Salesforce ISV ContentVersion best practices managed package"
> - "AppExchange security review file upload"
> - "Salesforce managed package trigger ContentDocumentLink"
> What FLS/CRUD checks needed? File type validation? Sharing model requirements?
>
> **Topic 7: Performance at Scale — Real-World Reports**
> - "Salesforce ContentDocumentLink query performance large org"
> - "Salesforce Files performance millions of records"
> - "Salesforce ContentVersion slow query"
> Any throttling? Query optimization strategies for ContentDocumentLink?
>
> ## Step 3: Write findings
>
> Save to: `MessageForge.Documentation/research/salesforce-files-deep-research.md`
>
> Format per topic:
> ```
> ## Topic N: [Title]
> ### Findings
> [Bullet points with source URLs]
> ### Confidence: HIGH/MED/LOW
> ### Solutions for Challenges Found
> [If any issue found — how to solve it]
> ```
>
> End with: **Unresolved Questions** (things needing scratch org testing)

---

### AGENT 2: Architecture & Solution Design

```
subagent_type: general-purpose
isolation: worktree
run_in_background: true
```

**Full prompt for Agent 2:**

> The decision to store ALL media in Salesforce ContentDocument/ContentVersion is FINAL. Do not recommend against it. Your job is to design the BEST architecture for this approach.
>
> # Task: Architecture Design — ContentVersion Media Storage
>
> Read existing docs, use WebSearch to fill gaps, produce the complete architecture design for ContentVersion-based media storage.
>
> ## Step 1: Read these files
> - `MessageForge.Documentation/architecture/architecture.md`
> - `MessageForge.Documentation/architecture/adr.md` (all ADRs, especially 3, 7, 8, 14)
> - `MessageForge.Documentation/reference/sf-data-model.md`
> - `MessageForge.Documentation/reference/governor-limits.md`
> - `MessageForge.Documentation/plans/appexchange-onboarding.md`
> - `MessageForge.Documentation/reference/security-checklist.md`
> - `MessageForge.Documentation/research/salesforce-contentdocument-media-storage.md`
> - `.claude/skills/media-pipeline.md`
> - `.claude/skills/data-ingestion.md`
>
> ## Step 2: Use WebSearch to verify technical details
> - ContentVersion REST API multipart upload format
> - ContentDocumentLink ShareType/Visibility best practices for ISV
> - LWC file display URL patterns (/sfc/servlet.shepherd/)
> - lightning-file-upload component capabilities and limits
> - NavigationMixin filePreview usage
>
> ## Step 3: Produce complete architecture design
>
> ### 3A: Updated System Architecture
> New data flow diagram (ASCII) showing:
> ```
> Telegram → Go Middleware → Salesforce REST API → ContentVersion
>                                                    ↓
>                                            ContentDocumentLink → Messenger_Message__c
>                                                    ↓
>                                            LWC (same-origin /sfc/servlet.shepherd/)
> ```
> And outbound:
> ```
> LWC (lightning-file-upload) → ContentVersion → Go downloads via REST → Telegram
> ```
>
> ### 3B: Complete Data Model Changes
> Exact field definitions (API name, type, length, default) for:
> - New fields on `Messenger_Message__c`: `Has_File__c`, `Contains_Inbound_Files__c`, `File_Count__c`
> - New fields on `Messenger_Attachment__c`: `Content_Version_Id__c`, `Content_Document_Id__c`, `Thumbnail_CV_Id__c`
> - What to do with `Media_URL__c`, `Thumbnail_URL__c` (deprecation strategy for managed package — can't delete fields)
>
> ### 3C: Inbound Flow Design (Telegram → SF)
> Step-by-step with exact API calls:
> 1. Go receives media from Telegram
> 2. Go uploads to SF: `POST /services/data/v62.0/sobjects/ContentVersion` (multipart, with FirstPublishLocationId)
> 3. What Go sends back to SF as message metadata (ContentVersionId, ContentDocumentId)
> 4. How the Messenger_Message__c and Messenger_Attachment__c records get the file references
> 5. How Has_File__c gets set (trigger vs Go sets it directly)
>
> ### 3D: Outbound Flow Design (SF → Telegram)
> Step-by-step:
> 1. Agent attaches file in LWC (lightning-file-upload or custom input)
> 2. ContentVersion created automatically
> 3. How Go gets notified (Platform Event? REST callback? Pub/Sub?)
> 4. Go downloads: `GET /services/data/v62.0/sobjects/ContentVersion/{id}/VersionData`
> 5. Go streams to Telegram
>
> ### 3E: LWC Display Design
> For each media type, specify:
> - **Photo**: thumbnail URL pattern, click-to-preview behavior, album grid layout
> - **Video**: `<video>` tag approach, poster image, what happens with large files (solution, not problem)
> - **Voice**: `<audio>` tag, waveform visualization approach
> - **Document**: file type icon (SLDS doctype:*), click-to-preview (NavigationMixin filePreview), download button
> - **Sticker**: image display, size constraints
>
> ### 3F: SOQL Query Design
> Efficient queries for chat thread with files:
> - How to load messages (existing query + new Has_File__c field)
> - How to batch-load files for messages with Has_File__c=true (minimize SOQL count)
> - Avoid N+1 problem
> - When to use @wire vs imperative
>
> ### 3G: Components Added/Removed/Modified
> Table showing every component that changes:
> | Component | Change | Details |
> |---|---|---|
> | Go: R2 adapter | REMOVE | Replace with SF ContentVersion adapter |
> | Go: presigned URL generator | REMOVE | No longer needed |
> | ... | ... | ... |
>
> ### 3H: ContentDocumentLink Trigger Design
> - Trigger on ContentDocumentLink (standard object, in managed package)
> - Logic: filter for Messenger_Message__c links, update Has_File__c + File_Count__c
> - Bulk-safe design (up to 200 records per trigger)
> - Governor limit considerations
>
> ## Step 4: Write to file
>
> Save to: `MessageForge.Documentation/research/salesforce-files-architecture-design.md`

---

### AGENT 3: Risk Solutions & Limits Workarounds

```
subagent_type: general-purpose
isolation: worktree
run_in_background: true
```

**Full prompt for Agent 3:**

> The decision to store ALL media in Salesforce ContentDocument/ContentVersion is FINAL. Do not recommend against it. Your job is to identify every risk and provide a SOLUTION or WORKAROUND for each. No risk is a blocker — every risk has a mitigation.
>
> # Task: Risk Register with Solutions — Salesforce Files Media Storage
>
> ## Step 1: Read these files
> - `MessageForge.Documentation/reference/governor-limits.md`
> - `MessageForge.Documentation/reference/security-checklist.md`
> - `MessageForge.Documentation/research/salesforce-contentdocument-media-storage.md`
> - `MessageForge.Documentation/architecture/adr.md`
> - `MessageForge.Documentation/plans/appexchange-onboarding.md`
> - If exists: `MessageForge.Documentation/reviews/2026-03-29-risk-register.md` (for format reference)
>
> ## Step 2: Use WebSearch to verify limits and find real-world solutions
>
> ## Step 3: Analyze 10 risks — EACH with a concrete SOLUTION
>
> For EACH risk:
> - **Scenario**: How it happens (concrete example)
> - **Math model** (where applicable): real numbers
> - **Impact**: What breaks
> - **SOLUTION**: How we solve it (this is the critical part — not "accept the risk" but "here's how we handle it")
> - **Implementation**: What code/config to write
>
> **R1: Storage Growth — Client Org Fills Up**
> Math: 100 users, 200 media msgs/day, avg 500KB/file = X GB/month
> Enterprise base: 10GB + 2GB/user = 210GB for 100 users
> Time to exhaust?
> SOLUTION REQUIRED: Storage monitoring from managed package, admin alerts, auto-cleanup policies, file retention settings
>
> **R2: API Call Budget — Heavy Media Usage**
> Math: each upload = 1 API call. Model: 50/500/5000 media/day
> Enterprise: 100K/day, Unlimited: varies
> SOLUTION REQUIRED: Queuing strategy, throttling on Go side, API budget monitoring, fallback queue when near limit, API add-on packages for clients
>
> **R3: Large Video Upload — Timeout/Memory**
> Scenario: 500MB video from Telegram, Go uploads to SF REST API
> Risk: timeout, memory pressure, Content-Length requirement
> SOLUTION REQUIRED: Streaming approach, temp file buffering, chunking strategy, timeout configuration, retry with resume
>
> **R4: ContentDocumentLink Trigger — Conflicts with Other Packages**
> Risk: other installed packages have triggers on same standard object
> SOLUTION REQUIRED: Defensive trigger pattern, error handling, governor-safe design, testing strategy
>
> **R5: AppExchange Security Review — File Upload Requirements**
> Risk: review rejection for missing FLS/CRUD checks
> SOLUTION REQUIRED: Exact checklist of security checks needed, code patterns that pass review
>
> **R6: Chat UI Performance — Many Images**
> Risk: 50+ images in chat thread, all loading at once
> SOLUTION REQUIRED: Lazy loading strategy, pagination, thumbnail renditions, virtual scrolling approach
>
> **R7: Video Playback — No Range Requests from SF Servlet**
> Risk: user can't seek in video, must download entirely first
> SOLUTION REQUIRED: UX pattern that works (progressive download, download button, thumbnail preview before play, small video auto-play + large video download-first)
>
> **R8: Managed Package Field Deprecation**
> Risk: can't delete Media_URL__c in package upgrade, stale fields
> SOLUTION REQUIRED: Deprecation strategy (mark unused, hide from layouts, keep for backward compat)
>
> **R9: Multi-Org Storage Variance**
> Risk: subscriber orgs have wildly different storage quotas
> SOLUTION REQUIRED: Runtime storage check API, admin dashboard showing usage, graceful degradation when near limit
>
> **R10: GDPR File Deletion**
> Risk: message deleted but ContentVersion persists
> SOLUTION REQUIRED: Cascade deletion design, configurable retention policy, admin setting for auto-delete-on-message-delete
>
> ## Step 4: Verified Limits Table
>
> Compile ALL relevant limits with sources:
> | Limit | Value | Source | Workaround if Hit |
> |---|---|---|---|
> | Max file (REST multipart) | ? | [url] | ... |
> | API calls/day (Enterprise) | ? | [url] | ... |
> | Apex heap size | ? | [url] | ... |
> | ... | ... | ... | ... |
>
> ## Step 5: Write to file
>
> Save to: `MessageForge.Documentation/research/salesforce-files-risk-solutions.md`
>
> Format: Executive summary → Risk register table → Detailed risk+solution per R1-R10 → Verified limits table

---

### AGENT 4: ADR-20 + Implementation Roadmap

```
subagent_type: general-purpose
isolation: worktree
run_in_background: true
```

**Full prompt for Agent 4:**

> The decision to store ALL media in Salesforce ContentDocument/ContentVersion is FINAL. This is an ACCEPTED ADR, not a proposal. Write it as decided.
>
> # Task: Write ADR-20 (Accepted) + Implementation Roadmap + Doc Update Plan
>
> ## Step 1: Read existing ADRs for format
> - `MessageForge.Documentation/architecture/adr.md` — match ADR-18/ADR-19 format exactly
> - `MessageForge.Documentation/architecture/architecture.md`
> - `MessageForge.Documentation/research/salesforce-contentdocument-media-storage.md`
> - `MessageForge.Documentation/plans/mvp-implementation-plan.md` — for phase structure
> - `MessageForge.Documentation/reviews/2026-03-29-implementation-roadmap.md` — for roadmap format
> - `.claude/skills/media-pipeline.md` — current pipeline being replaced
> - `.claude/skills/question-resolver.md` — has "Media in R2" principle to update
> - `CLAUDE.md` (root) — has "Never store media in Salesforce" rule to update
>
> ## Step 2: Use WebSearch for competitor validation
> - "SMS Ninja AppExchange file storage ContentVersion"
> - "Salesforce ISV managed package ContentVersion best practice"
>
> ## Step 3: Write ADR-20 (ACCEPTED status)
>
> Save to: `MessageForge.Documentation/architecture/adr-20-media-storage-pivot.md`
>
> Must include:
> - **Status:** Accepted
> - **Date:** 2026-03-30
> - **Supersedes:** ADR-3, ADR-14
> - **Context:** Business motivation (3 reasons), competitor validation
> - **Decision:** ALL media stored in Salesforce ContentDocument/ContentVersion. R2 removed entirely.
> - **Rationale:** Numbered list with evidence
> - **Technical approach:** Brief summary of ContentVersion/ContentDocumentLink/FirstPublishLocationId pattern
> - **Trade-offs accepted:** Higher client storage cost (feature not bug), no video range requests (workaround: download-first UX), API call consumption (monitoring + throttling)
> - **Consequences:** Components removed (R2, Workers, CSP, CORS, media_files PG table), components added (ContentVersion adapter, ContentDocumentLink trigger, new fields), docs to update
> - **Risks with mitigations:** Top 5 risks with solutions (not "accept risk" — actual solutions)
>
> Also append a SUMMARY LINE to the ADR table in the main `MessageForge.Documentation/architecture/adr.md`:
> ```
> | **ADR-20** | Salesforce ContentVersion for all media storage (supersedes ADR-3, ADR-14) | Client-side storage liability, AppExchange revenue alignment, SMS Ninja validates pattern. R2 eliminated. |
> ```
>
> ## Step 4: Write Implementation Roadmap
>
> Save to: `MessageForge.Documentation/plans/media-storage-migration-plan.md`
>
> Break into phases:
>
> **Phase 1: Foundation (data model + Go adapter)**
> - Add new fields: Has_File__c, Content_Version_Id__c, etc.
> - Build Go ContentVersion upload adapter (multipart)
> - Build Go ContentVersion download adapter (streaming)
> - Unit tests for Go adapter
>
> **Phase 2: Inbound Flow (Telegram → SF Files)**
> - Wire Go inbound pipeline to upload to ContentVersion instead of R2
> - Update inbound message processing to include ContentVersionId/ContentDocumentId
> - ContentDocumentLink trigger for Has_File__c
> - Integration tests
>
> **Phase 3: LWC Display (show files in chat)**
> - Update messengerChat to display images via /sfc/servlet.shepherd/ URLs
> - Video, voice, document, sticker display
> - Album grid layout
> - Lightbox/preview via NavigationMixin
> - Lazy loading
>
> **Phase 4: Outbound Flow (SF → Telegram)**
> - File upload in LWC (lightning-file-upload or custom)
> - Go download from ContentVersion + send to Telegram
> - Platform Event or callback for upload notification
>
> **Phase 5: Cleanup & Polish**
> - Remove R2 adapter code from Go
> - Remove R2/Workers infrastructure config
> - Remove CSP Trusted Sites for R2
> - Deprecate Media_URL__c fields (keep for upgrade safety, hide from layouts)
> - Update permission sets
> - Storage monitoring dashboard (admin LWC component)
>
> Each phase: list tasks, agent to use (go-backend / salesforce-dev), dependencies, verification command.
>
> ## Step 5: Write Doc Update Plan
>
> List every file that needs updating (but DON'T edit them — just list):
>
> | File | What Changes | Priority |
> |---|---|---|
> | `CLAUDE.md` (root) | "Never store media in Salesforce" → "All media in ContentVersion" | P0 |
> | `.claude/skills/media-pipeline.md` | Complete rewrite: R2 → ContentVersion | P0 |
> | `.claude/skills/question-resolver.md` | Principle #6 "Media in R2" → remove | P0 |
> | `MessageForge.Documentation/architecture/architecture.md` | Update data flow diagram, remove R2 | P1 |
> | `MessageForge.Documentation/reference/sf-data-model.md` | Add new fields | P1 |
> | `MessageForge.Documentation/reference/security-checklist.md` | Add ContentVersion CRUD/FLS checks | P1 |
> | `MessageForge.Salesforce/CLAUDE.md` | Remove "never store files in SF" | P0 |
> | ... | ... | ... |

---

## PHASE 2: Wait for Completion

After launching all 4 agents in Phase 1, wait for all to complete. You will be notified automatically when each finishes. Do NOT proceed until all 4 are done.

Track progress — as each completes, note what it produced.

---

## PHASE 3: Consolidate into Unified Document

Once all 4 agents complete, their output files may be in worktree branches. Check and merge:

1. Check if output files exist in main tree. If not, the worktree agents wrote them in branches — use `git` to get the content or read from worktree paths.

2. Read all output files:
   - `research/salesforce-files-deep-research.md` (Agent 1)
   - `research/salesforce-files-architecture-design.md` (Agent 2)
   - `research/salesforce-files-risk-solutions.md` (Agent 3)
   - `architecture/adr-20-media-storage-pivot.md` (Agent 4)
   - `plans/media-storage-migration-plan.md` (Agent 4)

3. Cross-reference for contradictions:
   - Agent 1 claims X about API limits, Agent 3 assumes different number → reconcile
   - Agent 2 designs a flow, Agent 3 identifies risk in that flow → verify solution exists
   - Agent 4's ADR mentions risks → verify they match Agent 3's solutions

4. Write FINAL consolidated document:

Save to: `MessageForge.Documentation/research/salesforce-files-FINAL-solution.md`

```markdown
# Salesforce Files Media Storage — Complete Solution Document

**Date:** 2026-03-30
**Decision:** ACCEPTED (ADR-20). All media in Salesforce ContentVersion.

## 1. Executive Summary
[3-5 sentences: what we're doing, why, how, key technical approach]

## 2. Architecture Design
[From Agent 2: data flow, data model changes, component changes]

## 3. Verified Technical Facts
[From Agent 1: web-verified API details, limits, competitor evidence]
[Table: Claim | Verified? | Source URL | Notes]

## 4. Risk Solutions
[From Agent 3: all 10 risks with their SOLUTIONS]
[No unsolved risks — every risk has a mitigation]

## 5. Implementation Roadmap
[From Agent 4: 5-phase plan with tasks and agents]

## 6. ADR-20 Reference
[Point to adr-20-media-storage-pivot.md]

## 7. Documentation Update Checklist
[From Agent 4: every file to update, with priority]

## 8. Verified Limits Table
[From Agent 3: all limits with sources and workarounds]

## 9. Unresolved Items (Scratch Org Testing Needed)
[Combined from all agents: things that can only be verified by testing]

## 10. Next Action
"The implementation prompt is ready at: research/agent-prompt-salesforce-files-migration.md"
"Start with Phase 1 of the migration plan: plans/media-storage-migration-plan.md"
```

5. If any agent's output file is missing or empty, note it in the consolidated doc and fill the gap yourself using the other agents' findings + your own research.

6. Verify quality:
   - [ ] Every risk from Agent 3 has a SOLUTION (not just "accept it")
   - [ ] Architecture from Agent 2 covers all media types (photo/video/voice/doc/sticker/album)
   - [ ] API details from Agent 1 have source URLs
   - [ ] ADR-20 from Agent 4 has ACCEPTED status (not PROPOSED)
   - [ ] Implementation roadmap has concrete phases with tasks
   - [ ] No agent recommended against the decision (remind: it's final)

---

## PHASE 4: Notify User

After consolidation, present a summary to the user:

```
## Salesforce Files Media Storage — Research Complete

**Decision:** ACCEPTED. All media in Salesforce ContentVersion/ContentDocumentLink.
**ADR-20:** Written and accepted (supersedes ADR-3, ADR-14).

### Key Findings
- [3-5 most important verified facts]

### Solutions for Top Challenges
- Video streaming: [solution found]
- API budget: [solution found]
- Storage growth: [solution found]

### Documents Produced
1. `research/salesforce-files-deep-research.md` — verified research
2. `research/salesforce-files-architecture-design.md` — complete architecture
3. `research/salesforce-files-risk-solutions.md` — 10 risks with solutions
4. `architecture/adr-20-media-storage-pivot.md` — accepted ADR
5. `plans/media-storage-migration-plan.md` — 5-phase implementation plan
6. `research/salesforce-files-FINAL-solution.md` — unified solution document

### Ready to Implement
Start with Phase 1 of the migration plan, or use the implementation prompt at `research/agent-prompt-salesforce-files-migration.md`.

### Items Needing Scratch Org Verification
- [list from unresolved questions]
```

---

## RULES

1. **This is GO. Not a go/no-go analysis.** The decision is made. Find solutions, not objections.
2. **Every risk needs a SOLUTION.** "Accept the trade-off" is only valid if you explain WHY it's acceptable with numbers.
3. **Use WebSearch aggressively.** Previous research had no web access — verify claims with live sources.
4. **Rate findings.** HIGH = official Salesforce docs. MED = community consensus. LOW = single source.
5. **NO code changes.** Documentation only. The implementation comes after.
6. **The final deliverable is the consolidated document** (`salesforce-files-FINAL-solution.md`).
7. **ADR-20 status is ACCEPTED**, not PROPOSED. The decision is made.
