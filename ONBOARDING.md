# PiyushBuildOS — Project Onboarding Guide

> **Who reads this:** The agent or human starting a new project, or taking over an existing one.
> **When:** Before any crons are created, before any code is touched.
> **Output:** A fully wired project where all agents know what to do, where to go, and who to talk to.

---

## Part A: New Project From Scratch

### Step 1: Gather Required Inputs

Before touching anything, collect all of these. Do not proceed until every item is resolved.

| Input | What it is | How to get it |
|---|---|---|
| **Project name** | Human-readable name | User provides |
| **Asana project GID** | The PM tool project identifier | Create project in Asana web UI → URL contains the GID, OR user provides existing |
| **GitHub org + repo URL** | Where the code lives | Create repo in GitHub OR user provides existing |
| **Local checkout path** | Where on this machine the repo lives | User specifies (e.g. `~/code/myproject`) |
| **4-checkout layout** | monorepo, requirements, dev, site — see below | Derive from local path |
| **LiteLLM proxy URL** | Where all models route through | From `~/.hermes/config.yaml` or user specifies |
| **Slack workspace + channels** | Where agents deliver alerts | User creates channels or provides IDs — see Step 3 |
| **Site publish path/URL** | Where docs and marketing site live | User specifies (GitHub Pages, Synology, custom) |
| **Apple Team ID** | If shipping iOS/macOS apps | From Apple Developer Portal |
| **ASC app IDs** | If shipping to TestFlight | From App Store Connect — must be created manually in web UI |

---

### Step 2: Initialize the 4-Checkout Layout

Every PiyushBuildOS project uses 4 isolated checkouts to prevent context pollution:

```bash
# 1. Main monorepo (Build Manager works here via worktrees)
git clone <repo-url> ~/code/<project>

# 2. Requirements branch (Requirements agent works here)
git clone <repo-url> ~/code/<project>-requirements
cd ~/code/<project>-requirements && git checkout -b requirements && git push -u origin requirements

# 3. Dev/interactive branch (Process Coordinator one-offs)
git clone <repo-url> ~/code/<project>-dev
cd ~/code/<project>-dev && git checkout -b dev && git push -u origin dev

# 4. Site repo (separate repo for docs/marketing)
git clone <site-repo-url> ~/code/<project>-site
```

**Stay in your lane:**
- Build Manager → `~/code/<project>` (main, via worktrees at `/tmp/<project>-build-{slug}`)
- Requirements agent → `~/code/<project>-requirements` (requirements branch)
- Process Coordinator → reads any, writes only via pipeline
- Site agent → `~/code/<project>-site`

---

### Step 3: Create Communication Channels

Minimum 3 Slack channels (or equivalent):

| Channel | Purpose | Who delivers |
|---|---|---|
| `#<project>-build` | Autonomous build alerts (RTB pickups, shipped, failed) | Build Manager cron |
| `#<project>-buildprocess` | Interactive oversight — Piyush's interface | Process Coordinator |
| `#<project>-requirements` | Requirements agent updates (delta-only) | Requirements cron |
| `#<project>-drift` | Daily drift audit results | Drift Audit cron |

Record channel IDs (not names) — Slack API uses IDs. Get them from channel settings.

---

### Step 4: Initialize Agent Context Files

Copy the role templates from `PiyushBuildOS/roles/` into your project's requirements branch:

```bash
cd ~/code/<project>-requirements
mkdir -p docs/agent-context
cp ~/code/PiyushBuildOS/roles/*.md docs/agent-context/
cp ~/code/PiyushBuildOS/roles/requirements-agent-state.json docs/agent-context/
```

Then fill in project-specific values in each file. At minimum, update `SHARED-CONTEXT.md` with:
- Project name, owner, GitHub org
- Asana project GID
- Slack channel IDs
- Repo layout (4 paths)
- LiteLLM proxy URL
- Site URL
- Asana tag GIDs (see Step 5)

---

### Step 5: Initialize Asana Tags

The pipeline depends on specific status tags. Create them in your Asana project:

**Required status tags:**
- `status-ready-to-plan`
- `status-ready-to-build`
- `status-in-progress`
- `status-ready-for-qa`
- `status-qa-passed`
- `status-docs-pending`
- `status-docs-done`
- `status-shipped`
- `status-released`
- `status-blocked`

**Required phase tags:**
- `phase-v1` (or your equivalent milestone)
- `phase-v2`

**Agent tags:**
- `agent-claude`, `agent-codex`, `agent-hermes`, `agent-opencode`

Record all tag GIDs and add them to `SHARED-CONTEXT.md`. Store in `/tmp/<project>_asana_meta.json` for runtime access.

---

### Step 6: Verify Model Access

Run the model verification checklist before creating any crons.
See `models/MODEL_VERIFICATION.md` for the full procedure.

**Minimum:** primary model responds, at least one fallback is confirmed reachable.

---

### Step 7: Create Cron Jobs

See `CRON_SETUP.md` for exact prompts and schedules.

**Required crons (in creation order):**
1. Requirements agent (every 20m)
2. Build Manager (every 10m)
3. Drift Audit (daily)

Optional but recommended:
4. Dashboard generator (every 10m)
5. Status page (every 30m)

---

### Step 8: Verify End-to-End

Create one test task in Asana, tag it `status-ready-to-plan`, and verify:
- [ ] Requirements agent picks it up and creates a feature branch
- [ ] Build Manager sees the `status-ready-to-build` tag
- [ ] Slack alerts land in the correct channels
- [ ] COORDINATION.md gets an entry

If any step fails, check `CRON_SETUP.md` § Verification.

---

## Part B: Taking Over an Existing Project

If the user provides a repo URL or Asana project that already has content:

### Step 1: Baseline Audit (read-only — do not touch anything yet)

```bash
# Audit git state
git clone <repo-url> ~/code/<project>
cd ~/code/<project>
git log --oneline -20              # Recent history
git branch -r                      # All branches
git tag --sort=-version:refname | head -10  # Releases
```

**Questions to answer before proceeding:**
1. What is the current MARKETING_VERSION / version string?
2. What is the last published release (GitHub releases, DMG, TestFlight)?
3. What branches exist? Are any feature branches abandoned mid-build?
4. What is the build toolchain? (Swift/Xcode, Node, Python, etc.)
5. Are there existing CI/CD pipelines (GitHub Actions, etc.)? Will they conflict with agents?

### Step 2: Asana Audit (if existing project provided)

```bash
# Enumerate all tasks and their current status tags
# Use Asana API: GET /projects/<GID>/tasks?opt_fields=name,tags
```

Map existing task states to the PiyushBuildOS tag model:
- What's done vs what's in flight vs what's planned?
- Are there any `status-in-progress` tasks with no active owner (stale)?
- Does git state match Asana state (S-S-D check)?

### Step 3: Document the Delta

Write a `BASELINE.md` in the repo root:
- Current version
- What's already shipped
- What was in progress when the agent took over
- Known issues inherited from previous development
- Any deviations from PiyushBuildOS process to be aware of

### Step 4: Reconcile S-S-D Model

| Check | How |
|---|---|
| Asana `status-released` tasks → verify they're in the latest release artifact | Compare task names against release notes / changelog |
| `status-shipped` tasks → verify they're in git main | `git log --oneline` search |
| Git main features → verify they have Asana tasks | Browse recent commits |
| Published artifacts → verify they match git tags | `gh release list` vs `git tag` |

Anything mismatched gets flagged in BASELINE.md and corrected before agents start.

### Step 5: Continue from Step 4 of Part A

Once baseline is clean, proceed: initialize context files, verify model access, create crons.

---

## Checklist Summary

```
□ All inputs gathered (Asana GID, GitHub URL, local paths, Slack channels, site URL)
□ 4-checkout layout initialized
□ Slack channels created and IDs recorded
□ Agent context files initialized from PiyushBuildOS/roles/ templates
□ SHARED-CONTEXT.md filled in with project-specific values
□ Asana tags created and GIDs recorded in SHARED-CONTEXT.md
□ Model access verified (see MODEL_VERIFICATION.md)
□ Cron jobs created (see CRON_SETUP.md)
□ End-to-end test task passed
□ [Takeover only] BASELINE.md written, S-S-D model reconciled
```
