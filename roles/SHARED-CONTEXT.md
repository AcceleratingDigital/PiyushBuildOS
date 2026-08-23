# hOS Shared Agent Context — Read First

> **Last updated:** 2026-08-22
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
| **Slack: #piyush-mm4p-buildprocess** | BuildProcessCoordinator + Piyush | Build status, release alerts, process commands from Piyush |
| **Telegram** | Requirements agent | Requirements agent updates (delta-only, silent when nothing moved) |
| **Asana task notes** | ALL agents | Persistent handoff data — specs, diagnostics, branch names, build notes |
| **COORDINATION.md** | BuildProcessCoordinator | Append-only build log (baton handoffs, stall detection) |

### Communication rules

1. **Delta-only:** Never report "no changes" or "everything is fine." Silence = healthy.
2. **Asana notes are the contract:** If it's not in Asana task notes, it didn't happen.
3. **Branch names go in Asana notes:** The requirements agent puts `feature/{slug}` in task notes before tagging `status-ready-to-build`. The coordinator reads it from there.
4. **Slack commands from Piyush** are equivalent to GUI chat commands. The coordinator honors both.

### Two session types for Piyush

| Session | Surface | Context File |
|---|---|---|
| **GUI (Hermes desktop)** | This chat | `build-process-coordinator.md` + this shared file |
| **Slack (#piyush-mm4p-buildprocess)** | Slack channel | Same context files — the coordinator agent reads them regardless of which surface the command came from |

Both sessions share the same agent context files, Asana project, and git repos.
A command from Slack and a command from the GUI have equal authority.

## 5. Repo Layout (4 Checkouts — Stay in Your Lane)

| Checkout | Branch | Who Works Here |
|---|---|---|
| `~/code/hos-monorepo` | main | BuildProcessCoordinator ONLY (builds in worktrees) |
| `~/code/hos-requirements` | requirements | Requirements agent + Piyush |
| `~/code/hos-dev` | dev | Hermes interactive sessions (code fixes, release process) |
| `~/code/hos-site` | main | Site agent (marketing site, doc pages, downloads) |

**Nobody commits directly to main except the coordinator merging feature PRs.**

## 6. Asana Tag System

### Status tags (pickup signals)
- `status-ready-to-plan` → Requirements agent picks up
- `status-ready-to-build` → BuildProcessCoordinator picks up (branch name MUST be in notes)
- `status-in-progress` → Active build (coordinator tracking)
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
| BuildProcessCoordinator | Hermes (this session) | `algolia/xlarge` | `claude-sonnet-4-6` |
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
→ package-release.sh → publish-release.sh → DMG drop to ~/Downloads/hermes/hos/
→ TestFlight upload (iPhone + iPad) → status-shipped → Human Testing
→ status-released (FINAL)
```

**Mandatory post-release step:** BuildProcessCoordinator MUST sync ALL tasks in that release
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
30 minutes, the coordinator reverts the task to `status-ready-to-build` and
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
