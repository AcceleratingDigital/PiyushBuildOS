# hOS Build Coordinator — Agent Context

> **Last updated:** 2026-08-16 15:00 CST
> **Purpose:** Persistent context for the build coordinator agent (this Hermes session).
> If this session crashes or restarts, read this file + the handoff doc + memory to recover.

## My Role

I am Piyush's **build coordinator** for hOS. I am SEPARATE from the requirements agent
(which Piyush is setting up in a different session). My responsibilities:

1. **Manage the coordinator cron** (`e4b0b407e0fb`, every 10m → telegram:8283210299)
2. **Pick up specced tasks** from the requirements agent — tasks must have:
   - `status-ready-to-build` tag
   - `feature/{slug}` branch on origin (name in Asana task notes)
   - Build Readiness metadata in Asana notes (dependencies, parallel-safe, risk, scope)
   - Scope doc at `docs/scope/{slug}.md` on the branch
3. **Dispatch build pipeline**: coder → QA → review → docs (auto-advance, silent intermediate steps)
4. **Merge PRs** to main (one PR per feature, specs+code+QA+docs together)
5. **Release pipeline** when Piyush says "ship vX.Y.Z"
6. **Daily drift audit** (cron `fbe086d5b979`, every 24h → Telegram)

## What I Do NOT Do

- Write feature specs (that's the requirements agent's job)
- Create feature branches (requirements agent does this at ready-to-plan)
- Work in `~/code/hos-requirements` (that's the requirements agent's checkout)
- Commit directly to main (only merge PRs)
- Run `xcodebuild test` (GUI launch — use `xcodebuild build -configuration Release`)
- Build more than 1 feature at a time (serialized, user preference)

## Repo Layout (3 Checkouts)

| Checkout | Branch | Owner | Purpose |
|---|---|---|---|
| `~/code/hos-monorepo` | `main` | Coordinator cron | Build queue, worktrees, merges to main |
| `~/code/hos-requirements` | `requirements/v0.9` | Requirements agent | Spec writing, scope docs, feature branches |
| `~/code/hos-dev` | `dev/interactive` | This session (me) | Code fixes, release process, site work |

## Pipeline Flow (Feature Branch Model)

```
Requirements agent creates feature/{slug} branch + spec + scope doc
    → tags status-ready-to-build (LAST step, after specs committed)
    → Coordinator picks up:
        1. Read branch name from Asana notes
        2. git worktree add /tmp/hos-build-{slug} feature/{slug}
        3. Dispatch coder agent (delegate_task or Claude Code CLI)
        4. Coder adds code ON TOP of specs already on branch
        5. QA → review → docs (all on same branch, auto-advance)
        6. One PR to main (specs + code + QA + docs together)
        7. Merge → tag status-shipped → ✅ Telegram
```

## Current State (2026-08-16 15:00 CST)

- **Git main:** `8a5eb97` (docs commit, not a feature ship)
- **Released:** v0.5.0 (DMG published)
- **On main not in DMG:** 5 UX fixes (setup wizard, focus state, checkbox alignment, fallback provider, Foundation Models detection)
- **Feature branches on origin:** NONE (requirements agent hasn't created any yet)
- **10 status-ready-to-build tasks:** ALL unspecced (20-66 char notes, no branches, no Build Readiness)
  - These need the requirements agent to write proper specs + create branches
  - Coordinator will SKIP them until properly specced
- **11 stale status-qa-passed tasks:** From pre-feature-branch pipeline (pre-Aug 16)
  - These need evaluation: review+merge to main, or re-eval under new process
- **2 blocked:** Approval Flow (integration task, needs re-spec), M1 deploy test
- **1 ready-to-plan:** v2 Notes Read (concurrent pipe reads)
- **Decisions:** A1-A6, B1-B4 answered in DECISIONS-BEFORE-BUILD.md. Rest still open.

## Unspecced RTB Tasks (need requirements agent)

1. 1217507946790314 — Recovery / new-machine bootstrap
2. 1217508004833092 — App-managed privileged install (SMAppService)
3. 1217508004803803 — Chrome history read skill
4. 1217507686960086 — Obsidian integration (read)
5. 1217508016668848 — Document extraction (PDFs, scans)
6. 1217508016566422 — Reminders write (behind approval)
7. 1217508004303150 — CSV importer (Indian banks — ICICI/HDFC)
8. 1217507946202771 — Contacts write (behind approval)
9. 1217508004110966 — Calendar-time schedules v2
10. 1217507692011338 — Memory v1 — local store, per-member scoping

## Stale QA-Passed Tasks (11 — need review/merge decision)

1. 1217508608010758 — Journal — Obsidian read skill (1195c notes — has spec)
2. 1217508016412459 — Calendar write (behind approval)
3. 1217507881077736 — Mail triage + summarize
4. 1217508004447875 — Draft reply / RSVP
5. 1217507881074516 — mail-compose-send (write skill)
6. 1217507703993811 — Approval broker + in-app approval cards
7. 1217507686231706 — Single-use parameter-bound tokens
8. 1217507686158185 — Draft-then-confirm at skill level
9. 1217508468701330 — MCP mutate → 'parked' response (0c notes)
10. 1217508289760547 — Draft-then-confirm at skill level (doc 21) (0c notes)
11. 1217507686826861 — Morning system status composite

## Key Decisions Made (from DECISIONS-BEFORE-BUILD.md)

- **A1:** Beta = one complete vertical (Mail triage → Daily Brief → approval → loop queue)
- **A2:** Open Loops = build full for beta (core differentiator, agent-managed)
- **A3:** Daily Brief = full agent-synthesized (Agent Loop produces it, not Morning Status)
- **A4:** Finance = second vertical (categorization + anomaly + agent integration)
- **A5:** Full 5-tab iOS app for beta + Personal Agent Directives
- **A6:** Self-service installer for beta
- **B1:** Shared Capabilities Architecture = design FIRST, before any content skill
- **B2:** Memory = SQLite + NLEmbedding (on-device, free, no Postgres)
- **B3:** KB trigger as platform pattern — all skills emit knowledge from day 1
- **B4:** Multi-member = keep current root FDA helper (already works)

## Still Open Decisions

- B5 (XPC), B6 (contract versioning), C1-C5 (competitive), D1-D4 (sequencing),
  E1-E4 (risk/trust), F2-F8 (missing items), G1-G5 (research direction)

## Key Files

- **Process contract:** `docs/28-change-checklist.md`
- **Decisions:** `docs/DECISIONS-BEFORE-BUILD.md`
- **Coordination log:** `hos-server/docs/COORDINATION.md`
- **Watchdog state:** `docs/pipeline-stats/watchdog-state.json` (gitignored)
- **Scope docs:** `docs/scope/{slug}.md` (on feature branches)
- **Agent context files:** `docs/agent-context/*.md` (coder, qa, reviewer-*, docs-tech, marketing)
- **0.9 scope:** `hos-server/docs/RELEASE-0.9-SCOPE.md`

## Agent CLIs

| Agent | Binary | Model | Key flag |
|---|---|---|---|
| Coder | `claude` (v2.1.233) | algolia/xlarge via LiteLLM | `--dangerously-skip-permissions` |
| QA | `opencode` (v1.2.27) | algolia/medium via LiteLLM | `-m litellm/<model>` |
| Review | `codex` (v0.147.0) | gpt-5.6-sol via OpenAI | `--skip-git-repo-check` |
| Docs | `claude` | algolia/xlarge via LiteLLM | `--dangerously-skip-permissions` |

All CLIs verified present at `/opt/homebrew/bin/`.

## Telegram Alert Rules

ONLY: 🚧 started, ✅ shipped, 🚨 failed, 🚀 released. NO intermediate step alerts.

## Build Constraints

- ONE BUILD AT A TIME (serialized)
- Worktree-isolated: `/tmp/hos-build-{slug}`
- NEVER `xcodebuild test` (GUI launch) — use `xcodebuild build -configuration Release`
- SHARED FILES: COORDINATION.md, SKILL_MANIFEST, site/status.html, site/releases/index.html,
  docs/agent-context/*.md, docs/28-change-checklist.md, project.pbxproj
- ALWAYS `git diff --stat main..HEAD` before merge — revert unauthorized shared file changes
- macOS build passing ≠ iOS compiles — test both schemes

## LLM Vault

Keychain service `com.acceleratingdigital.hos`. Keys: llm-provider-url, llm-api-key,
llm-default-model, llm-fallback-model. If empty, chat silently fails.
Currently: LiteLLM proxy at 10.1.2.13:4000/v1, algolia/xlarge + algolia/medium.
