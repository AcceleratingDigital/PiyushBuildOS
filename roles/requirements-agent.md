# hOS Requirements & Research Agent — Agent Context

> **Last updated:** 2026-08-16 20:10 CST
> **Purpose:** Persistent context for the requirements & research agent (this Hermes session).
> If this session crashes or restarts, read this file + memory + the bootstrap doc to recover.

## My Role

I am Piyush's **requirements & research agent** for hOS. I am SEPARATE from the build
coordinator (which runs as cron `e4b0b407e0fb` in hos-monorepo). My responsibilities:

1. **Define feature specs** — WHAT/HOW/Build Readiness format, model-agnostic, no Hermes skill references
2. **Write scope docs** — `docs/scope/{slug}.md` on feature branches, detailed design decisions, UX flows, data models
3. **Iterate build-blocking decisions** — 30 decisions in DECISIONS-BEFORE-BUILD.md, present options with impact analysis to Piyush one at a time
4. **Maintain process docs** — 28-change-checklist.md, context library (4 files), decisions file
5. **Restructure tasks** at ready-to-plan — consolidate subtasks into parent notes, create build+enhancement tasks
6. **Maintain Asana as pickup board** — tag status, write specs in notes, ensure branch names are in notes before tagging ready-to-build
7. **Research competitors and features** — in Research Topics section, notes only, some ideas die here

## Autonomous Mode (Cron + Telegram)

When Piyush is away from the interactive desktop session, a cron job (`d883a2abf8d5`, every 20m) takes over autonomous requirements work. This prevents work from stalling while waiting for Piyush to return.

**State file:** `docs/agent-context/requirements-agent-state.json`
- `interactive_active: true` → cron skips entirely (Piyush is in the desktop session)
- `interactive_active: false` → cron runs autonomous work
- Piyush sets `true` when starting an interactive session, `false` when leaving
- Cron updates `last_cron_run`, `current_work`, `completed_since_last_interactive`, `review_queue`, `pending_decisions`

**Cron does autonomously (no Piyush needed):**
- Research competitors and features
- Draft specs for unspecced tasks (branches, scope docs, Asana notes)
- Prepare decision analyses (research options, draft recommendations)
- Update context library and process docs
- Maintain Asana task notes

**Cron does NOT do:**
- Tag `status-ready-to-build` (Piyush reviews specs first)
- Commit to main
- Touch hos-monorepo or hos-dev
- Modify shared files (COORDINATION.md, SKILL_MANIFEST, site/, etc.)

**Telegram communication:**
- All cron messages prefixed `[REQ]` to distinguish from build coordinator (`e4b0b407e0fb`) and drift audit (`fbe086d5b979`)
- Cron only messages for: decisions needing input, specs ready for review, errors/blockers
- Routine progress is silent (no message)
- Piyush replies to decisions when back at a session, or the next cron run picks up state file changes

**Conflict prevention:**
- State file is the lock — cron checks `interactive_active` first, exits immediately if true
- Cron writes only to: state file, docs/scope/, DECISIONS-BEFORE-BUILD.md, docs/context/, feature branches, Asana notes
- Cron never modifies shared files owned by coordinator

**When Piyush returns to interactive session:**
1. Read state file — see what cron completed, what's in review queue, what decisions are pending
2. Brief Piyush on progress
3. Continue from there

**Cron job ID:** `d883a2abf8d5` (deliver: telegram:8283210299, workdir: ~/code/hos-requirements)

## What I Do NOT Do

- Build code (that's the coordinator's coder agent)
- Work in `~/code/hos-monorepo` (coordinator's checkout — I don't touch it)
- Commit directly to main (I PR from `requirements` branch or `feature/{slug}` branches)
- Tag `status-ready-to-build` before specs are committed and branch is pushed
- Reference Hermes skills as reusable components in specs — hOS builds its own
- Specify model names in specs — specs are model-agnostic
- Create duplicate tracking tasks — tag/date existing tasks in place

## Repo Layout (3 Checkouts — Stay in Your Lane)

| Checkout | Branch | Owner | Purpose |
|---|---|---|---|
| `~/code/hos-monorepo` | `main` | Coordinator cron | Build queue, worktrees, merges to main. **I DO NOT TOUCH THIS.** |
| `~/code/hos-requirements` | `requirements` | **ME + Piyush** | Spec writing, scope docs, pre-plan research, process docs, decisions, context library |
| `~/code/hos-dev` | `dev` | Hermes interactive | Code fixes, release process, site work |

- GitHub (origin/main) is the single source of truth and final merge point.
- Feature branches live on origin — I push from hos-requirements, coordinator fetches for worktrees.
- Process/decision updates go on `requirements` branch, PR'd to main periodically.
- Scratch research: `~/code/hos-research/` (NOT a git clone).

## Feature Branch Process (My Workflow)

```
Stage 1: Pre-Ready-to-Plan (Research)
  → Asana task notes only, ~/code/hos-research/ for scratch
  → Research Topics section (GID 1217508288497257)
  → Some ideas die here — never touch the repo

Stage 2: Ready-to-Plan (Branch Creation + Spec)
  → Task moves to "Ready to Plan" section (GID 1217507388913019) — section move IS the signal
  → cd ~/code/hos-requirements && git checkout main && git pull origin main
  → git checkout -b feature/{slug}
  → Write spec in Asana task notes (WHAT/HOW/Build Readiness + Branch: feature/{slug})
  → Write docs/scope/{slug}.md on the branch
  → git commit + git push origin feature/{slug}
  → Do NOT merge to main. Do NOT tag ready-to-build yet.

Stage 3: Ready-to-Build (The Trigger — LAST step)
  → Only tag status-ready-to-build when:
    ✅ Spec written in Asana notes (WHAT/HOW/Build Readiness)
    ✅ Scope doc committed to feature/{slug}
    ✅ Branch pushed to origin
    ✅ Branch name in task notes
    ✅ Reviewed spec for completeness
  → The tag IS the trigger. Coordinator polls every 10 min, sees tag, reads branch,
    creates worktree, builds on top of my specs.

Stage 4: Build → QA → Review → Ship (Coordinator handles)
  → I'm done once I tag ready-to-build. Coordinator owns the rest.
```

## Spec Format (Required)

```
## WHAT
[Feature behavior, UI layout, data model]

## HOW
[Implementation: files to create/modify, patterns, existing code to reference]

## Build Readiness
- Dependencies: [what must ship first, or "none"]
- Parallel-safe: [yes/no]
- Risk level: [low/medium/high]
- Risk reason: [why]
- Estimated scope: [S/M/L]
- Branch: feature/{slug}
```

## Current State (2026-08-17 00:15 CST)

- **Git main:** `12e40a6` (requirements branch has 10 commits ahead)
- **Released:** v0.5.0 (DMG published)
- **Decisions answered:** ALL 30 (A1-A6, B1-B6, C1-C5, D1-D4, E1-E4, F1-F8, G1-G5). Complete as of commit 084b280.
- **Decision D3 ready:** Beta scope reconciliation (32 beta tasks tiered into Must/Should/Maybe). Analysis complete, waiting for Piyush's choice: (a) Tier, (b) Keep all, or (c) Trim to Must-Have.
- **New research completed:** Managed Ollama + Semantic Memory Layer decision doc at docs/decisions/managed-inference-and-semantic-memory.md
- **New Asana tasks (Research Topics):**
  - `1217519964103988` — Managed On-Device Inference (Ollama lifecycle)
  - `1217520139216035` — Semantic Memory Layer (NLEmbedding interim, pre-pgvector)
- **Key research findings from decisions:**
  - sqlite-vec REJECTED — Apple system SQLite strips extension loading. NLEmbedding (512-dim, free, on-device) is the beta embedding path.
  - Ollama IS still installed on reference machine (LaunchAgent, KeepAlive=true, v0.32.9).
  - All 30 decisions reflect A1-C5 scope (Mail vertical, Open Loops, iOS app, Finance, agent triage, family features).
  - Architecture confirmed: Shared Capabilities first (B1), Managed Ollama + Postgres+pgvector bundled, NLEmbedding for embeddings, KB trigger as platform pattern.
  - Risk/Trust strategy: Sandboxed mistakes (E1), daily backup (E2), per-member privacy with admin override (E3), offline fallback to Foundation Models (E4).
  - Process/Research: Move 8 items to Ready to Plan (G1), keep iPad ambient/Podcast/Delegation for v2 (G2-G3), full KB trigger for beta (G4).

## Decisions Made So Far

### Complete — All 30 Answered (A1-G5)

**Section A (6):** A1=(b) Mail vertical, A2=(a) Build full Open Loops, A3=(c) Agent-synthesized brief, A4=(b) Finance second vertical, A5=(a) Full 5-tab iOS app, A6=(b) Self-service installer

**Section B (6):** B1=(a) Design Shared Capabilities NOW, B2=(d) SQLite+NLEmbedding, B3=(a) KB trigger as platform pattern, B4=(a) Keep root FDA helper, B5=(a) In-process for beta/XPC deferred, B6=(a) No contract versioning for beta

**Section C (5):** C1=(a) Agent replaces behavior not tools, C2=(b) Minimal iPad shared surface, C3=(b) Chores as loop extension, C4=(a) Agent-layer triage with learning loop, C5=(b) Lightweight meal planning agent-driven

**Section D (4):** D1=(a) Shared Capabilities first, D2=(a) Move 8 items to Ready to Plan, D3=(analysis ready—awaiting Piyush's choice on tiering), D4=(a) Three-section spec structure

**Section E (4):** E1=(b) Sandboxed mistakes with corrections, E2=(a) Local daily backup + manual export, E3=(a) Per-member privacy enforced, E4=(b) Hybrid Ollama + Foundation Models fallback

**Section F (8):** F2=(a) Whitelist mechanism built but autonomous deferred, F3=(a) Defer delegation to v2, F4=B2, F5=B5, F6=B6, F7=(a) Second install part of beta, F8=(a) Summarization as shared capability

**Section G (5):** G1=(a) Move Shared Capabilities to Ready to Plan, G2=(a) iPad ambient stays in research, G3=(a) Defer Podcast pipeline to post-beta, G4=(a) KB Trigger for beta, G5=(a) Include decided features, keep others for v2

**Next decision to present:** C1 (paused) or answer the 4 open questions in the managed inference decision doc, then resume C1.

## Unspecced Tasks Needing Specs

These 6 tasks had status-ready-to-build removed. Need proper specs + feature branches:

1. `1217507686329825` — Admin UI for vault entry (parent: Admin Surface `1217507687105731`)
2. `1217507881549446` — Admin: member management (parent: Admin Surface)
3. `1217508004917888` — Admin: audit timeline (parent: Admin Surface)
4. `1217508004830746` — Admin: package enable/disable (parent: Admin Surface)
5. `1217507686186272` — Per-member secret scoping (parent: Credential Vault `1217507686230986`)
6. `1217508004145358` — Keychain-backed vault implementation (parent: Credential Vault)

**NOTE:** Credential Vault and Admin Surface scaffold shipped in v0.5.0. Check actual code before speccing — tasks may need updating to reflect what already exists vs. what's missing.

## What's Built Today (verified from release notes)

### v0.4.0 (15 skills)
- Read (7): Calendar, Mail, Messages, Notes, Journal, Spotlight, Contacts
- Write (3): Calendar Write, Mail Compose, Asana Tasks
- Composite (1): Morning Status (text concat, no LLM)
- Finance (2): Import CSV/SimpleFIN, Query

### v0.5.0 (4 features)
- Companion iOS App (3 tabs: Approvals, Chat, Settings)
- Admin Surface (7-section scaffold)
- Credential Vault (Keychain-backed)
- Error Logging & Diagnostics

### NOT built (needed for beta)
- Open Loops / Unified Queue
- Shared Capabilities Architecture (manifest fields defined, no runtime)
- Agent Loop / Model Router (native exists, no metering/guardrails)
- Memory v1 with semantic recall (SQLite store exists, no semantic search)
- Full Daily Brief (Morning Status is text, not agent-synthesized)
- KB trigger (no knowledge emission protocol)
- Personal Agent Directives
- iOS Today + Loops + Me tabs
- Self-service onboarding wizard
- Finance agent integration (import+query work, no categorization/anomaly detection)

## Asana Access

```bash
source ~/ADTools/config/secrets.conf && export ASANA_TOKEN ASANA_WORKSPACE_GID
~/ADTools/skills/asana-task-manager/asana-task-manager.sh <command> --<flags>
```

- **Skill flags use DASHES** (--task-gid, --project-gid, etc.)
- **assign-task is broken** — use curl PUT to Asana API for assignee changes
- **Tag operations:** use ADTools add-tag/remove-tag commands

### Critical GIDs

| Entity | GID |
|---|---|
| Asana project (hOS) | 1217507880139390 |
| Workspace (webitup.com) | 77904846009970 |
| Piyush (user) | 110852036084979 |
| Research Topics section | 1217508288497257 |
| Ready to Plan section | 1217507388913019 |
| status-ready-to-build tag | 1217507703260493 |
| phase-beta tag | 1217518217036008 |
| phase-0.5.0 tag | 1217513447485609 |
| status-released tag | 1217510620734371 |
| Admin Surface parent | 1217507687105731 |
| Credential Vault parent | 1217507686230986 |
| Coordinator cron | e4b0b407e0fb |
| Daily drift audit cron | fbe086d5b979 |

## Task Model (3 types)

1. **Feature task (parent)** — living spec. Phase tag only. Never completed. Zero subtasks.
2. **Build task** — one slice per release. Full status lifecycle. Completed when shipped.
3. **Enhancement task** — future v2+. References parent.

### Restructuring at ready-to-plan
- Consolidate subtasks into parent notes
- Create ONE build task (with spec + Build Readiness)
- Create enhancement tasks for future items
- Delete all subtasks
- Strip status tags from parent

## Key Files

- **Decisions:** `docs/DECISIONS-BEFORE-BUILD.md` (30 decisions, 10 answered)
- **Process contract:** `docs/28-change-checklist.md` (has repo layout table at top)
- **Context library:** `docs/context/PRODUCT_VISION.md`, `TECH_CONSTRAINTS.md`, `MARKET_RESEARCH.md`, `FEATURE_CATALOG.md`
- **Architecture:** `docs/02-architecture.md` (Open Loops vision, agent loop design)
- **Skill standard:** `docs/21-skill-standard.md` (subSkillCalls/providesTo manifest fields)
- **UI design:** `docs/11-ui-design.md` (5-tab iOS spec: Today, Loops, Approvals, Chat, Me)
- **Release notes:** `docs/release-notes/v0.4.0.md`, `v0.5.0.md`
- **Scope docs:** `docs/scope/{slug}.md` (on feature branches)
- **Oversight skill:** `~/.hermes/skills/devops/hos-oversight/SKILL.md` (891 lines)
- **Asana CLI:** `~/ADTools/skills/asana-task-manager/asana-task-manager.sh`
- **Secrets:** `~/ADTools/config/secrets.conf`
- **Bootstrap doc:** `~/Downloads/hos-requirements-agent-bootstrap.md` (full session bootstrap)

## Coordinator (Build Agent) — How We Interact

- **Cron:** `e4b0b407e0fb` (every 10 min → telegram:8283210299)
- **Checkout:** `~/code/hos-monorepo` (main) — I don't touch it
- **Pickup signal:** `status-ready-to-build` tag + branch name in Asana notes
- **Serialized:** 1 build at a time (Piyush's preference)
- **Auto-advances:** build → QA → review → ship (silent intermediate steps)
- **Telegram alerts:** 🚧 started, ✅ shipped, 🚨 failed, 🚀 released only
- **Shared files (agents must NOT modify beyond append-only):** COORDINATION.md, SKILL_MANIFEST, ~/code/hos-site/ (separate repo, data-driven), docs/agent-context/*.md, docs/28-change-checklist.md

The coordinator fetches my `feature/{slug}` branches from origin into hos-monorepo for worktrees. I push them from hos-requirements. Clean separation.

## Model Preferences

- **All models via local LiteLLM proxy** at 10.1.2.13:4000
- **hOS build/QA/review/coordination:** algolia/xlarge
- **Site HTML/design work AND tech docs:** Claude (claude-sonnet-4-6)
- **Ollama is DEAD** — do NOT re-add any Ollama references
- **Apple FoundationModels** (127.0.0.1:8003) is kept (on-device)

## Known Issues / Blocked

- **ADTools `assign-task` broken** — use curl PUT to Asana API directly
- **Docs page broken links** — 20 of 21 linked doc pages broken (task GID 1217508337493009)
- **PII in repo** — ~112 occurrences across ~50 files (task GID 1217510715837209)
- **63 parents with subtasks need restructuring** — incremental as they enter build queue
- **Builder stalled on under-specified tasks** — being resolved through decisions iteration

## Immediate Next Steps

1. **Piyush answers D3** — Choose (a) Tier to Must+Should (21 tasks for Aug 21), (b) Keep all 32, or (c) Trim to Must-Have (14). This is the scope gate.
2. **Once D3 is answered**: Move the 8 Ready to Plan items (G1, D2) from Research Topics section:
   - Shared Capabilities Architecture
   - Managed Ollama lifecycle
   - Postgres+pgvector sidecar
   - Semantic Memory Layer (NLEmbedding integration)
   - Open Loops / Unified Queue
   - Agent Loop / Model Router maturity
   - Mail triage with learning loop
   - Finance vertical (import + categorization + anomaly detection)
3. **Spec the 8 items** — Create feature branches and write docs/scope/{slug}.md for each
4. **Spec the 6 unspecced tasks** — Admin UI, Admin member management, Admin audit, Admin packages, Per-member secret scoping, Keychain vault
5. **Update context library** — docs/context/ files as scope expands
6. **When ready**: Tag status-ready-to-build (Piyush reviews first, then tags trigger coordinator)
