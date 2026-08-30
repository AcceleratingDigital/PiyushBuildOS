# Project Configuration Template

> **Who fills this in:** Process Coordinator at project onboarding.
> **Where it goes:** Copy to `docs/agent-context/SHARED-CONTEXT.md` in your project's
> requirements branch. This template has placeholder values — replace ALL of them.
> Delete this header block when done.

---

# [PROJECT NAME] Shared Agent Context — Read First

> **Last updated:** [DATE]
> **Purpose:** Shared context file that ALL agents must read at session start.
> Each agent's individual context file builds on top of this.

---

## 1. Project Identity

- **Product:** [Product name and one-line description]
- **Owner:** [Name] ([email])
- **Company:** [Company name] (Apple Team ID: [TEAM_ID] if applicable)
- **GitHub Org:** [org name]
- **Current Phase:** [e.g. Beta, v1, v2]

---

## 2. The S-S-D Model (Single-Sourced Data)

| Surface | Source of Truth | What Lives Here |
|---|---|---|
| **Intent** | Asana (project GID [ASANA_PROJECT_GID]) | Task definitions, specs, decisions, status tags |
| **Build** | Git (origin/main) | Code, branches, PRs, release tags |
| **Reality** | [Published artifact URL / location] | What users actually install and experience |

**Never let these drift.** If a task is `status-released` in Asana but the artifact
doesn't contain it, that's a critical inconsistency.

---

## 3. Current Version

- **MARKETING_VERSION:** Check `[path to version file in repo]` — do NOT hardcode
- **Next build:** Determined by Build Manager at release time
- **Release notes:** `[path to release notes in repo]`
- **Release process:** `[path to release checklist]`

---

## 4. Communication Channels

| Channel | Who uses it | Purpose |
|---|---|---|
| **[PLATFORM]: #[project]-build** (ID: [CHANNEL_ID]) | Build Manager (cron [CRON_ID]) | Autonomous build alerts |
| **[PLATFORM]: #[project]-buildprocess** (ID: [CHANNEL_ID]) | Process Coordinator + [Owner] | Interactive oversight |
| **[PLATFORM]: #[project]-requirements** (ID: [CHANNEL_ID]) | Requirements agent | Requirements updates |
| **[PLATFORM]: #[project]-drift** (ID: [CHANNEL_ID]) | Drift Audit cron | Daily drift alerts |

**Communication rules:**
1. Delta-only: Never report "no changes." Silence = healthy.
2. Asana notes are the contract: If it's not in Asana task notes, it didn't happen.
3. Branch names go in Asana notes before tagging `status-ready-to-build`.

---

## 5. Repo Layout (4 Checkouts — Stay in Your Lane)

| Checkout | Path | Branch | Who Works Here |
|---|---|---|---|
| Monorepo | `[LOCAL_PATH]` | main | Build Manager + Process Coordinator ONLY |
| Requirements | `[LOCAL_PATH]-requirements` | requirements | Requirements agent + [Owner] |
| Dev | `[LOCAL_PATH]-dev` | dev | Interactive sessions, one-off fixes |
| Site | `[LOCAL_PATH]-site` | main | Site agent |

**Nobody commits directly to main except the Build Manager merging feature PRs.**

---

## 6. Asana Tag System

### Status tags (pickup signals)
- `status-ready-to-plan` GID: [GID] → Requirements agent picks up
- `status-ready-to-build` GID: [GID] → Build Manager picks up
- `status-in-progress` GID: [GID] → Active build
- `status-ready-for-qa` GID: [GID] → QA agent picks up
- `status-qa-passed` GID: [GID] → Reviewer picks up
- `status-docs-pending` GID: [GID] → Docs agent picks up
- `status-docs-done` GID: [GID] → Ready for release
- `status-shipped` GID: [GID] → On main, NOT in artifact yet
- `status-released` GID: [GID] → In published artifact (FINAL)
- `status-blocked` GID: [GID] → Stuck, needs attention

### Phase tags
- `phase-v1` GID: [GID]
- `phase-v2` GID: [GID]

### Agent tags
- `agent-claude` GID: [GID]
- `agent-codex` GID: [GID]
- `agent-hermes` GID: [GID]
- `agent-opencode` GID: [GID]

**Tag GIDs stored at:** `/tmp/[project]_asana_meta.json` — regenerate if missing.

---

## 7. Tool & Model Matrix

| Role | CLI Tool | Primary Model | Fallback |
|---|---|---|---|
| Build Manager (cron [CRON_ID]) | Hermes (cron) | [MODEL] | [FALLBACK_MODEL] |
| Process Coordinator (interactive) | Hermes (this session) | [MODEL] | [FALLBACK_MODEL] |
| Requirements (cron [CRON_ID]) | Hermes (cron) | [MODEL] | [FALLBACK_MODEL] |
| Coder | `claude` CLI | [MODEL] | [FALLBACK_MODEL] |
| QA | `opencode` CLI | [MODEL] | [FALLBACK_MODEL] |
| Reviewer (security) | `codex` CLI | [MODEL] | [FALLBACK_MODEL] |
| Reviewer (performance) | `codex` CLI | [MODEL] | [FALLBACK_MODEL] |
| Docs/Site | `claude` CLI | [MODEL] | [FALLBACK_MODEL] |

**LiteLLM proxy:** `[PROXY_URL]`
**Fallback chain:** If primary fails → fallback → alert Process Coordinator if fallback also fails.
**Model verification:** Run `models/MODEL_VERIFICATION.md` before first build.

---

## 8. Release Pipeline

```
status-ready-to-build → status-in-progress → status-ready-for-qa → status-qa-passed
→ status-docs-pending → status-docs-done → [RELEASE GATE]
→ [project-specific release steps: version bump, build, sign, publish]
→ SMOKE TEST: launch artifact, verify no crash (10s). If crash → STOP.
→ [Platform-specific distribution: DMG / TestFlight / npm / PyPI / etc.]
→ verify all artifacts → status-shipped → Human Testing
→ status-released (FINAL — requires human confirmation)
```

**Mandatory smoke test:** Every release must verify the artifact runs without crashing BEFORE distribution. If it crashes: STOP. File bug, fix, rebuild.

**Mandatory multi-platform rule:** [List all platforms this project ships to — skip none.]

---

## 9. Concurrency Guardrails

### One build at a time
Build Manager serializes builds. Only one `status-in-progress` task at a time.

### Worktree isolation
All builds happen in `/tmp/[project]-build-{slug}` worktrees. Never in the main checkout.

### Human developer collision
- **Branch protection:** `main` requires PR review — no direct pushes (enforced in GitHub).
- **In-flight detection:** Before starting a build, check `git fetch` for commits on the feature branch since the Requirements agent last pushed. If found: a human touched the branch — read changes before proceeding.
- **Yield protocol:** If a human has an open PR to the same branch, hold the automated build and alert Process Coordinator.

### Baton pattern
When passing work to another agent, write to COORDINATION.md:
- **Who** is receiving the baton
- **What** task + branch + commit SHA
- **What state** it's in

### Stale lease detection
`status-in-progress` is a lease. If no COORDINATION.md heartbeat for 30 minutes, Build Manager reverts to `status-ready-to-build` and posts a stall alert.

---

## 10. Known Issues & Technical Debt

[Fill in as the project accumulates known issues. Example format:]
- **[Issue name]:** [Description]. Fix deferred to [milestone]. Asana: [task GID].

---

## 11. What Each Agent Should Do With This File

1. **Read it at session start** (before your role-specific context file)
2. **Check the `Last updated` date** — if more than 3 days old, check COORDINATION.md for unrecorded events before proceeding
3. **Update it when shared state changes**
4. **Do NOT put role-specific instructions here** — those go in your `<role>.md`

## 12. Cross-Role Boundary Table

| Role | Cannot do |
|---|---|
| **Process Coordinator** | Edit source files; run build commands; push directly to main; merge PRs; fix bugs during testing |
| **Build Manager** | Edit source files; write feature specs; create feature branches; run more than 1 build at a time |
| **Coder** | Merge PRs to main; change Asana tags beyond status-ready-for-qa; modify shared files without coordinator awareness |
| **QA** | Accept coder self-reported audit results without independent verification; mark qa-passed without running the build |
| **Requirements Agent** | Pick up status-in-progress tasks; work in the main monorepo checkout |
| **ALL agents** | Skip context file updates after significant events; declare audits complete without independent verification |

## 13. Human Confirmation Gate

Before `status-released` on any task requiring human validation (device launch, UX approval, etc.):
1. Artifact distributed to test target ✅
2. Human installs and tests ✅
3. Human explicitly confirms: "[confirmation phrase for this project] v[X.Y.Z]" ✅

Build Manager cannot self-apply the final confirmation — it requires the project owner.
