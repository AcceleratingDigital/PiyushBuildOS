# Collision Model — Multi-Agent & Multi-Human Concurrency

> **Who reads this:** Build Manager, Process Coordinator, Requirements agent.
> **Why it matters:** Agents and humans working the same repo simultaneously without
> coordination causes merge conflicts, overwritten work, and broken builds.
> This document defines the detection, avoidance, and recovery protocols.

---

## 1. Agent-Agent Collision (within PiyushBuildOS)

### One Build at a Time (serialized)
The Build Manager only runs one feature build at a time. Before picking up a new
`status-ready-to-build` task, verify:

```bash
# No other active worktrees
ls /tmp/<project>-build-*/  2>/dev/null && echo "ACTIVE BUILD" || echo "CLEAR"

# No other status-in-progress tasks
# Query Asana: GET /tasks?tag=<status-in-progress-GID>&project=<PROJECT_GID>
```

If an active build is found: wait for it to complete. Do NOT start a new one.

### Worktree Isolation
All builds happen in isolated worktrees:
```bash
git worktree add /tmp/<project>-build-{slug} feature/{slug}
```
Never work directly in `~/code/<project>` — that's the main checkout for reads only.

### Checkout Lane Rules
| Agent | Checkout | Branch |
|---|---|---|
| Build Manager | `~/code/<project>` (reads) + `/tmp/<project>-build-*` (writes via worktrees) | main |
| Requirements | `~/code/<project>-requirements` | requirements |
| Process Coordinator | reads any; never writes source | N/A |
| Site Agent | `~/code/<project>-site` | main |

**Crossing lanes = collision risk.** The Requirements agent never touches the monorepo main checkout. The Build Manager never touches the requirements checkout.

### Baton Pattern (no dropped handoffs)
Every agent-to-agent handoff is written to COORDINATION.md:
```
[TIMESTAMP] [FROM AGENT] → [TO AGENT]: [task GID] | branch: feature/{slug} | commit: [SHA] | state: [description]
```
No silent handoffs. If it's not in COORDINATION.md, the baton was dropped.

### Stale Lease Detection
`status-in-progress` is a time-bounded lease (30 min). Build Manager checks:
- Is there a COORDINATION.md entry within the last 30 min for this task?
- If no: assume stale, revert to `status-ready-to-build`, post stall alert.

---

## 2. Agent-Human Collision (humans also working the repo)

This is the most common source of unexpected conflicts in team projects.

### Branch Protection (enforce in GitHub)

Set on `main`:
```
✅ Require pull request before merging
✅ Require at least 1 approval
✅ Dismiss stale reviews when new commits are pushed
✅ Require status checks (if CI exists)
❌ Allow force pushes — NEVER
❌ Allow deletions — NEVER
```

This prevents both humans AND agents from pushing directly to main.
Only the Build Manager merges PRs — and only after QA + review pass.

### Detecting Human Activity on a Feature Branch

Before starting a build on `feature/{slug}`, the Build Manager checks:

```bash
git fetch origin
git log --oneline origin/feature/{slug} ^$(git merge-base main origin/feature/{slug})
```

If commits exist beyond what the Requirements agent created:
- A human touched the branch
- Read those commits before proceeding
- If the commits complete the spec: hand directly to QA, skip coder
- If the commits partially complete it: brief the coder on what's done
- If the commits conflict with the spec: flag to Process Coordinator, hold build

### Open Human PRs

Before the Build Manager opens an automated PR, check for existing PRs on the same branch:

```bash
gh pr list --repo <org>/<repo> --head feature/{slug}
```

If a human PR is already open:
- Do NOT open a second PR
- Review the human PR for spec compliance
- If compliant: trigger QA on that PR's branch, use the human's PR
- If not compliant: comment on the PR with gaps, hold automated build

### When a Human Pushes to Main Directly

This should not happen if branch protection is set. If it does (admin bypass):
1. Build Manager detects it on next poll via `git fetch` + `git log origin/main`
2. Flags in COORDINATION.md and Slack alert
3. Checks if any in-flight feature branch is now behind main (needs rebase)
4. Rebases affected branches before continuing

---

## 3. Multi-User Projects (team of humans + agents)

When multiple humans are working the same project alongside agents:

### What Agents Monitor
- `git fetch` before every build cycle — detect new commits on tracked branches
- Open PRs — agents never auto-merge a PR another human has open reviews on
- Asana task assignments — if a task is assigned to a human, agents skip it
- CI status — if CI is red on main, Build Manager holds all new merges until green

### Yield Protocol
When the Build Manager detects human activity that could conflict:
1. Post to `#<project>-buildprocess`: "⚠️ Human activity detected on [branch/area]. Holding [task] until clear. Reply 'resume' to continue."
2. Pause the affected build (do not kill the worktree — preserve state)
3. Wait for Process Coordinator (Piyush) to reply "resume" or "abort"

### Human-Assigned Tasks
If an Asana task has `agent-piyush` tag OR is assigned to a named user (not an agent):
- Build Manager skips it entirely
- Requirements agent skips it
- Only Process Coordinator monitors it and asks the human for status

---

## 4. Resource Contention

### Postgres / Database
If the project runs a local database:
- Only one worktree should run migrations at a time
- Build Manager checks for active worktrees before running `migrate`
- Test databases use worktree-specific port numbers or temp directories

### Build Tools (Xcode, etc.)
- Only one `xcodebuild archive` at a time (macOS constraint)
- Build Manager serializes all archive operations
- Never run two archives concurrently even if the feature branches are different

### Network / API Rate Limits
- LiteLLM proxy enforces per-model rate limits upstream
- If a model returns 429: retry with exponential backoff (max 3 attempts), then fall back
- Log rate limit hits in COORDINATION.md so the pattern is visible over time

---

## 5. Recovery Protocols

### Stale Worktree
```bash
# Check for stale worktrees (no corresponding in-progress task)
git worktree list

# Remove if stale
git worktree remove /tmp/<project>-build-{slug} --force
git branch -D feature/{slug}  # only if coder never pushed
```

### Merge Conflict on Feature Branch
When a feature branch has conflicts with main after main advanced:
```bash
cd /tmp/<project>-build-{slug}
git fetch origin
git rebase origin/main

# If conflicts: resolve them
# For shared files (COORDINATION.md, version file): always append, never overwrite
# For source files: prefer the feature branch's version (it has spec-compliant code)
```

### Deadlocked Build (in-progress for >2h)
1. Read COORDINATION.md for last known state
2. Check if the coder agent process is still running
3. If dead: revert task to `status-ready-to-build`, clean worktree, restart
4. If alive but stuck: steer the agent (if interactive) or kill + restart (if cron)
5. Post incident summary to COORDINATION.md

---

## Summary Checklist (per build cycle)

```
□ No other active worktrees for this project
□ No other status-in-progress tasks  
□ git fetch: no unexpected commits on feature branch
□ No open human PRs on same feature branch
□ CI green on main (if CI exists)
□ No human-assigned tasks being picked up
□ Database/build tool resources available
```
