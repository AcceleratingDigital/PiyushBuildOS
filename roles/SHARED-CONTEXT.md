# hOS Shared Agent Context — Read First

> **Last updated:** 2026-08-24
> **Purpose:** Shared context file that ALL agents (coordinator, requirements,
> coder, QA, reviewers, docs, marketing, human-testing) must read at session
> start. Contains cross-cutting information that affects every role.
>
> **Each agent's individual context file** (`<role>.md`) builds on top of this.
> If this file and your role file conflict, YOUR ROLE FILE wins for role-specific
> behavior, but THIS FILE wins for shared state, communication, and process rules.

---

## 1. Project Identity

- **Product:** hOS — a family operating system (Mac Server + iPhone/iPad/Watch companions)
- **Owner:** Piyush Patel (patelpk@me.com)
- **Company:** webitup LLC (Apple Team ID: 4KCNX5MRR5)
- **GitHub Org:** AcceleratingDigital
- **Current Phase:** Beta (phase-beta tag in Asana)

## 2. The S-S-D Model (Single-Sourced Data)

| Surface | Source of Truth | What Lives Here |
|---|---|---|
| **Intent** | Asana (project GID 1217507880139390) | Task definitions, specs, decisions, status tags |
| **Build** | Git (origin/main) | Code, branches, PRs, release tags |
| **Reality** | Published DMGs + TestFlight + marketing site | What users actually install and experience |

**Never let these drift.** If a task is `status-released` in Asana but the DMG
doesn't contain it, that's a critical inconsistency. If code is on main but
isn't tagged `status-shipped`, the board is lying.

## 3. Current Version

- **MARKETING_VERSION:** Check `project.pbxproj` — do NOT hardcode version numbers in your context
- **Next build:** Determined by coordinator at release time
- **Release notes:** `docs/release-notes/vX.Y.Z.md`
- **Release process:** `docs/28-change-checklist.md` § "Release packaging and publishing"

## 4. Communication Channels

### Where agents talk

| Channel | Who uses it | Purpose |
|---|---|---|
| **Slack: #piyush-mm4p-hosbuild** | Build Manager (cron e4b0b407e0fb) | Autonomous build alerts — RTB pickups, shipped, failed, released |
| **Slack: #piyush-mm4p-hosbuildprocess** | Process Coordinator + Piyush | Interactive — Piyush's oversight, status checks, process improvement |
| **Telegram** | Requirements agent | Requirements agent updates (delta-only, silent when nothing moved) |
| **Asana task notes** | ALL agents | Persistent handoff data — specs, diagnostics, branch names, build notes |
| **COORDINATION.md** | Build Manager + Process Coordinator | Append-only build log (baton handoffs, stall detection) |

### Communication rules

1. **Delta-only:** Never report "no changes" or "everything is fine." Silence = healthy.
2. **Asana notes are the contract:** If it's not in Asana task notes, it didn't happen.
3. **Branch names go in Asana notes:** The requirements agent puts `feature/{slug}` in task notes before tagging `status-ready-to-build`. The coordinator reads it from there.
4. **Slack commands from Piyush** are equivalent to GUI chat commands. The coordinator honors both.

### Two session types for Piyush

| Session | Surface | Context File |
|---|---|---|
| **GUI (Hermes desktop)** | This chat | `process-coordinator.md` + this shared file |
| **Slack (#piyush-mm4p-hosbuildprocess)** | Slack channel (Process Coordinator) | `process-coordinator.md` + this shared file — Piyush's interactive oversight |
| **Slack (#piyush-mm4p-hosbuild)** | Slack channel (Build Manager cron) | `build-manager.md` + this shared file — autonomous build alerts |

Both session types share the same agent context files, Asana project, and git repos.
A command from Slack and a command from the GUI have equal authority.

## 5. Repo Layout (4 Checkouts — Stay in Your Lane)

| Checkout | Branch | Who Works Here |
|---|---|---|
| `~/code/hos-monorepo` | main | Build Manager + Process Coordinator ONLY (builds in worktrees) |
| `~/code/hos-requirements` | requirements | Requirements agent + Piyush |
| `~/code/hos-dev` | dev | Hermes interactive sessions (code fixes, release process) |
| `~/code/hos-site` | main | Site agent (marketing site, doc pages, downloads) |

**Nobody commits directly to main except the coordinator merging feature PRs.**

## 6. Asana Tag System

### Status tags (pickup signals)
- `status-ready-to-plan` → Requirements agent picks up
- `status-ready-to-build` → Build Manager picks up (branch name MUST be in notes)
- `status-in-progress` → Active build (Build Manager tracking)
- `status-ready-for-qa` → QA agent picks up
- `status-qa-passed` → Reviewer picks up
- `status-docs-pending` → Docs agent picks up
- `status-docs-done` → Ready for release
- `status-shipped` → On main, NOT in DMG yet
- `status-released` → In published DMG/TestFlight (FINAL state)
- `status-blocked` → Stuck, needs attention
- `status-needs-design` → Research backlog

### Phase tags
- `phase-beta` — Must ship for beta
- `phase-v1` — Post-beta V1
- `phase-v2` — Future

### Agent tags
- `agent-claude`, `agent-codex`, `agent-hermes`, `agent-opencode`, `agent-piyush`, `agent-docs`

**Tag GIDs are in `/tmp/hos_asana_meta.json`. If that file is missing, regenerate it.**

## 7. Tool & Model Matrix

| Role | CLI Tool | Primary Model | Escalation |
|---|---|---|---|
| Build Manager (cron e4b0b407e0fb) | Hermes (cron) | `algolia/xlarge` | `claude-sonnet-4-6` |
| Process Coordinator (interactive) | Hermes (this session) | `algolia/xlarge` | `claude-sonnet-4-6` |
| Requirements | Hermes (cron d883a2abf8d5) | `algolia/xlarge` | `claude-sonnet-4-6` |
| Coder | `claude` CLI | `algolia/xlarge` via LiteLLM | `claude-sonnet-4-6` |
| QA | `opencode` CLI | `algolia/medium` | `algolia/xlarge` |
| Reviewer (security) | `codex` CLI | `gpt-5.6-sol` | — |
| Reviewer (performance) | `codex` CLI | `algolia/xlarge` | `claude-sonnet-4-6` |
| Reviewer (user-value) | `codex` CLI | `algolia/xlarge` | `claude-sonnet-4-6` |
| Docs/Tech | `claude` CLI | `claude-sonnet-4-6` | `algolia/xlarge` |
| Marketing/Site | `claude` CLI | `claude-sonnet-4-6` | `algolia/xlarge` |
| Human Testing | Hermes (interactive) | `algolia/xlarge` | — |

**LiteLLM proxy:** `10.1.2.13:4000` (all models route through here)
**Public URL:** `https://llm.acceleratingdigital.com`
**All CLIs use the proxy via env vars (ANTHROPIC_BASE_URL, etc.)**

## 8. Release Pipeline (All Agents Must Know)

```
status-ready-to-build → status-in-progress → status-ready-for-qa → status-qa-passed
→ status-docs-pending → status-docs-done → [RELEASE GATE]
→ bump version + package-release.sh + publish-release.sh
→ DMG drop to ~/Downloads/hermes/hos/
→ SMOKE TEST: mount DMG, launch app, verify no crash (10s). If crash → STOP, file bug, rebuild.
→ TestFlight upload (iPhone Companion + iPad — ALL platforms, every release)
→ verify all builds VALID → status-shipped → Human Testing
→ status-released (FINAL)
```

**Mandatory smoke test rule:** EVERY release must launch the app on mm4p and
verify it runs without crashing for 10 seconds BEFORE publishing to TestFlight
or syncing Asana tags. If the app crashes: STOP. File a bug task (status-blocked),
fix, rebuild, re-test. No release ships without passing the smoke test.

**Mandatory multi-platform rule:** EVERY release ships ALL apps — Mac DMG,
iPhone Companion (TestFlight), and iPad "hOS Shared View" (TestFlight).
Watch app ships embedded in the iPhone Companion. No platform is skipped.
Release is not complete until all TestFlight builds reach VALID state.

**Mandatory post-release step:** Build Manager MUST sync ALL tasks in that release
from `status-shipped` → `status-released`. This is not optional. If tasks remain
at `status-shipped` after a DMG is published, the Asana board is lying to Piyush.

## 9. Concurrency Guardrails (Anti-Lock)

### Baton Pattern (handoffs)
When passing work to another agent, write an explicit handoff entry:
- **Who** is receiving the baton
- **What** task + branch + commit SHA
- **What state** it's in (spec done, code done, QA passed)

### Lock files (filesystem safety)
Before writing to a shared worktree or file, check for a `.lock` file. If one
exists, back off and alert the coordinator.

### Lease-based statuses (stall detection)
`status-in-progress` is a lease, not a permanent state. If no heartbeat for
30 minutes, the Build Manager reverts the task to `status-ready-to-build` and
posts a stall alert.

## 10. Known Issues & Technical Debt

- **Misspelling:** `AcceleratingDIgital` (capital I) used consistently across
  all bundle IDs, CloudKit container, and Apple Developer Portal. Do NOT
  fix piecemeal — it's a post-beta migration task (Asana 1217739078981956).
- **iPad entitlements:** Now fixed for production (APS + CloudKit container).
- **Watch app icons:** Now have 1024x1024 icon. Was empty before v0.6.4.
- **Postgres binaries:** Now signed in package-release.sh. Was failing notarization.

## 11. What Each Agent Should Do With This File

1. **Read it at session start** (before your role-specific context file)
2. **Update it when shared state changes** (new version, new known issue, process change)
3. **Do NOT put role-specific instructions here** — those go in your `<role>.md`
4. **Reference it from your role file** so new agents know to read it first

## 12. Cross-Surface Sharing Rules (MANDATORY — Read This Carefully)

**This is the #1 cause of past chaos.** Agents learn things in one session
(cron Build Manager discovers a build issue, coder finds a pitfall, QA catches
a regression) but the other surface — the interactive Process Coordinator,
the GUI session, the other cron — has no idea. Context files are committed to
the `requirements` branch, but running agents don't re-read them mid-session,
and there's no automatic push between surfaces.

### The Write-Back Rule

When you learn something that affects ANY other agent or surface, you MUST
write it to a shared location BEFORE finishing your turn:

| What you learned | Where to write it | Who needs it |
|---|---|---|
| Cross-cutting discovery (build issue, process flaw, new known issue) | **SHARED-CONTEXT.md** § 10 or relevant section | ALL agents |
| Task-specific finding (bug diagnosis, build result, QA verdict) | **Asana task notes** | Next agent in pipeline |
| Build state change (started, shipped, blocked, released) | **COORDINATION.md** + Slack alert | Build Manager, Process Coordinator |
| Role-specific learning (pattern, pitfall, convention) | **Your own `<role>.md`** | Future runs of your role |
| Process change (new guardrail, tag change, pipeline step) | **SHARED-CONTEXT.md** + affected `<role>.md` files | ALL agents |

### The Sync Note (every role file must have this)

Every agent context file (`<role>.md`) MUST contain this header block:

```
> **SYNC NOTE:** This file is shared between the Hermes desktop chat session
> AND any connected Slack channel for this role. Both surfaces read and write
> to it. Update it at every significant event so both stay aligned.
> Learnings that affect OTHER roles go to SHARED-CONTEXT.md, NOT just here.
```

### What "writing back" means in practice

1. **Cron agents (Build Manager, Requirements, Drift Audit):** After every run
   where you learned something, update the relevant shared file. Do NOT just
   accumulate learnings in your own context file silently.

2. **Interactive agents (Process Coordinator, Human Testing):** When you
   discover something during a session, write it to SHARED-CONTEXT.md or Asana
   notes immediately — do not wait for the session to end.

3. **Coder/QA/Reviewer agents:** Write your findings to Asana task notes (for
   the pipeline) AND to your own `<role>.md` (for future runs). If the finding
   is cross-cutting (affects other roles), also update SHARED-CONTEXT.md.

4. **All agents:** If you change a process, guardrail, or convention, update
   SHARED-CONTEXT.md in the SAME session. Do not leave it for "next time."
   Stale context files caused the v0.6.4/v0.6.5 crash chain — the Build Manager
   knew about the CKContainer crash but the interactive session didn't.

---

## Event-Driven Pipeline (active as of 2026-09-05)

The build pipeline now uses **direct chaining** instead of cron-poll dispatch. Each agent fires the next stage upon completion via `trigger.sh`.

### Trigger Script
```bash
~/ADTools/skills/hos-pipeline-trigger/trigger.sh \
  --task-gid GID --branch BRANCH --scope-doc DOC \
  --completed-step STEP --next-step STEP --summary "TEXT"
```
Advances Asana tag, updates `watchdog-state.json`, appends to `COORDINATION.md`, launches next agent.

### Failure Script
```bash
~/ADTools/skills/hos-pipeline-trigger/fail.sh \
  --task-gid GID --branch BRANCH --failed-step STEP \
  --reason "TEXT" --needed "TEXT"
```
Tags `status-blocked`, updates `watchdog-state.json`, posts structured Slack alert to `C0BRKHLDB7Z`.

### Pipeline Envelope Variables (injected by trigger.sh)
| Variable | Meaning |
|---|---|
| `HOS_TASK_GID` | Asana task GID |
| `HOS_BRANCH` | Feature branch |
| `HOS_SCOPE_DOC` | Scope doc path (relative to hos-monorepo) |
| `HOS_COMPLETED_STEP` | Previous completed stage |
| `HOS_NEXT_STEP` | Current agent's role |
| `HOS_SUMMARY` | What the previous agent did |
| `HOS_STARTED_AT` | Unix timestamp — stage start (watchdog stuck detection) |

### watchdog-state.json Schema (event-driven format)
Location: `docs/pipeline-stats/watchdog-state.json` (gitignored)
```json
{"task_gid":"...","branch":"feature/slug","scope_doc":"docs/scope/slug.md",
 "current_step":"qa","step_started_at":1757000000,"pipeline_started_at":1756990000,
 "completed_steps":["requirements","coder"],"summary":"...","paused":false}
```

### Stage Map
`requirements → coder → qa → reviewer-security → docs → coordinator-merge → release`

Auto-merge is enabled: PRs merge when security reviewer passes (no human gate for pipeline-triggered merges). Piyush's Slack ID `U3JHVDV2T` is the only authorized command sender for the monitor watchdog.

### Docs Tag GIDs (discovered 2026-09-05)
- `status-docs-pending`: `1217508540828634`
- `status-docs-done`: `1217508540826497`
