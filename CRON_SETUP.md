# Cron Setup Guide

> **Who reads this:** Process Coordinator at project onboarding or when rebuilding cron infrastructure.
> **Prerequisites:** Model access verified (see `models/MODEL_VERIFICATION.md`),
> agent context files initialized (see `ONBOARDING.md`), Slack channels created.

---

## Required Crons (in creation order)

Create in this order — the Requirements agent must exist before the Build Manager
tries to pick up tasks it creates.

### 1. Requirements Agent

**Purpose:** Triage unspecced tasks, write scope docs, create feature branches, tag RTB.
**Schedule:** Every 20 minutes (staggered: :05, :25, :45)

```
hermes cron create \
  --name "<project>-requirements" \
  --schedule "*/20 * * * *" \
  --model [MODEL] \
  --provider [PROVIDER] \
  --deliver "[SLACK_CHANNEL_ID]"
```

**Prompt template** (fill in project-specific values):
```
You are the Requirements Agent for [PROJECT NAME]. Your job is to pick up unspecced
tasks from the Asana project (GID: [ASANA_PROJECT_GID]) and turn them into
buildable specs.

READ FIRST: ~/code/[PROJECT]-requirements/docs/agent-context/SHARED-CONTEXT.md
Then read: ~/code/[PROJECT]-requirements/docs/agent-context/requirements-agent.md

PRIMARY DUTY: Bug triage and spec writing.
- Find tasks tagged phase-[PHASE] with NO status tag (or status-ready-to-plan)
- Investigate the codebase to understand scope
- Write spec in Asana task notes (WHAT/HOW/Build Readiness)
- Create feature/{slug} branch in ~/code/[PROJECT]-requirements
- Push branch
- Tag status-ready-to-build (LAST step)
- Add branch name to Asana task notes

If you cannot understand a task: tag status-blocked, explain in notes.
Delta-only output. If nothing changed: output "No changes." and stop.
```

**Verify:** After first run, check Slack channel for output. If "No changes." appears — correct (silent when idle). If it crashes, check model access.

---

### 2. Build Manager

**Purpose:** Pick up RTB tasks, dispatch coders, manage QA/review/docs pipeline, merge PRs, cut releases.
**Schedule:** Every 10 minutes

```
hermes cron create \
  --name "<project>-build-manager" \
  --schedule "*/10 * * * *" \
  --model [MODEL] \
  --provider [PROVIDER] \
  --deliver "[BUILD_SLACK_CHANNEL_ID]"
```

**Prompt template:**
```
You are the Build Manager for [PROJECT NAME]. You run the automated build pipeline.

READ FIRST: ~/code/[PROJECT]-requirements/docs/agent-context/SHARED-CONTEXT.md
Then read: ~/code/[PROJECT]-requirements/docs/agent-context/build-manager.md
Then read: ~/code/[PROJECT]/hos-server/docs/COORDINATION.md (last 50 lines)

YOUR ROLE: Orchestrator only. You dispatch coders, QA, reviewers. You do NOT
write source code. You do NOT commit directly to main. You ONLY merge PRs.

Each cycle:
1. Check for status-in-progress tasks (stale lease >30min → revert + alert)
2. Check for status-ready-to-build tasks → pick up ONE, dispatch coder
3. Check for status-ready-for-qa tasks → dispatch QA
4. Check for status-qa-passed tasks → dispatch reviewers → dispatch docs
5. Check for status-docs-done tasks → merge PR → tag status-shipped → Slack alert
6. Check for release command from [OWNER] → run full release pipeline

Before every build: verify no active worktrees (collision check).
Delta-only output. Silence = healthy pipeline.
```

**Stagger:** Run at :00, :10, :20, :30, :40, :50 — offset from Requirements (:05, :25, :45) to avoid simultaneous Asana writes.

---

### 3. Drift Audit

**Purpose:** Daily check that Asana intent, git build state, and published reality stay in sync.
**Schedule:** Daily (recommend: 6am local time, before workday)

```
hermes cron create \
  --name "<project>-drift-audit" \
  --schedule "0 6 * * *" \
  --model [MODEL] \
  --provider [PROVIDER] \
  --deliver "[DRIFT_SLACK_CHANNEL_ID]"
```

**Prompt template:**
```
You are the Drift Audit agent for [PROJECT NAME]. Run a daily S-S-D consistency check.

READ FIRST: ~/code/[PROJECT]-requirements/docs/agent-context/SHARED-CONTEXT.md

CHECK 1 — Asana vs Git:
- Tasks tagged status-released → verify their features exist in the latest release artifact
- Tasks tagged status-shipped → verify their commits are on git main
- Tasks tagged status-in-progress for >4h → flag as stale

CHECK 2 — Git vs Reality:
- Latest git tag → verify matching release artifact exists (GitHub release / DMG / etc.)
- Latest published artifact → verify version matches MARKETING_VERSION in [VERSION_FILE]

CHECK 3 — Context file freshness:
- Check Last updated date on SHARED-CONTEXT.md
- If >3 days: flag for Process Coordinator to update

Output: only report DRIFT (mismatches). If all checks pass: output "No drift detected." and stop.
Deliver to [DRIFT_SLACK_CHANNEL_ID].
```

---

### 4. Dashboard Generator (optional but recommended)

**Purpose:** Generates a live HTML dashboard of pipeline state from Asana.
**Schedule:** Every 10 minutes

```
hermes cron create \
  --name "<project>-dashboard" \
  --schedule "*/10 * * * *" \
  --model [MODEL] \
  --provider [PROVIDER] \
  --deliver "local"
```

---

## Cron Stagger Reference

Avoid running multiple crons at the same second to prevent Asana API conflicts:

```
:00  Build Manager
:05  Requirements Agent (first tick)
:10  Build Manager
:20  Build Manager
:25  Requirements Agent (second tick)
:30  Build Manager
:40  Build Manager
:45  Requirements Agent (third tick)
:50  Build Manager
```

---

## Verifying Crons Are Healthy

### List all crons
```bash
hermes cron list
```

### Check last run output
```bash
hermes cron log <cron-id>
```

### Force a manual run
```bash
hermes cron run <cron-id>
```

### Expected behavior by cron

| Cron | Healthy (idle) | Healthy (active) | Unhealthy |
|---|---|---|---|
| Requirements | "No changes." | Task moved to RTB, Slack message | Error / no output |
| Build Manager | Silence | Slack: "🚧 BUILD STARTED..." or "✅ shipped" | Error / no output |
| Drift Audit | "No drift detected." | Lists mismatches | Error / no output |

---

## Pausing / Resuming Crons

When Piyush is doing manual work that could conflict with automated builds:

```bash
# Pause
hermes cron pause <build-manager-cron-id>
hermes cron pause <requirements-cron-id>

# Resume when clear
hermes cron resume <build-manager-cron-id>
hermes cron resume <requirements-cron-id>
```

**Process Coordinator should proactively offer to pause when:**
- Piyush says he's doing manual work in the repo
- A device is connected for testing (avoid simultaneous TestFlight uploads)
- A release is being manually verified (smoke test window)

---

## Updating Cron Prompts

When process changes (new step, new gate, new guardrail):

```bash
hermes cron edit <cron-id>
```

Update the prompt to reflect the change. Also update the corresponding role context file
(`docs/agent-context/build-manager.md` or `requirements-agent.md`) in the same session.

**Both must stay in sync.** A cron prompt that diverges from the context file causes the
cron to follow outdated instructions while the interactive agent follows the new ones.
