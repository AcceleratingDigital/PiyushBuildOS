# PiyushBuildOS — Agentic Build Process Framework

This folder contains the consolidated process, roles, and context for the "Piyush-style" agentic build-out of complex software systems. This is a general-purpose framework derived from the hOS project, designed to be reused for new projects.

## Core Philosophy: The "S-S-D" Model (Single-Sourced Data)
The fundamental rule of this process is that every piece of information has exactly one source of truth to prevent drift and hallucination.
- **Product Intent (The "What"):** Managed in a Project Management tool (e.g., Asana). This is the a-priori truth for requirements, specs, and acceptance criteria.
- **Build State (The "How/Status"):** Managed in Git. Branch names, commit history, and state files (`STATE.md`, `PLAN.md`) are the evidence of progress.
- **User-Facing Truth:** Managed via a dedicated Marketing/Docs site. No feature is "Shipped" until its documentation exists.

## The 4-Checkout Layout (Role Isolation)
To prevent "context pollution" and accidental edits, different agents work in isolated checkouts:
1. **Monorepo/BuildProcessCoordinator:** (Main branch) Only the BuildProcessCoordinator agent commits here. Handles merges, worktrees, and final releases.
2. **Requirements:** (Requirements branch) Where the Requirements Agent + Human design specs and create feature branches.
3. **Dev/Interactive:** (Dev branch) For rapid prototyping, one-off fixes, and interactive sessions.
4. **Site:** (Standalone repo) Dedicated to documentation and marketing.

## The Agentic Pipeline (The "Loop")
The process moves from abstract idea to shipped feature through a strict, gated pipeline:

### 1. Intake & Research
- **Research Session:** Investigation of competitors, feasibility, and "vertical slice" definition.
- **Signal:** Moving a task to "Ready to Plan" is the trigger for the next phase.

### 2. Requirements & Spec (The "Contract")
- **Branching:** Requirements agent creates a `feature/{slug}` branch.
- **Design Layer:** The **UX Designer Agent** reviews the "WHAT" and produces a UX Design Doc (User journeys, layout, micro-copy) before the build starts.
- **Spec Standard:** Every feature must follow the 3-section format:
    - **WHAT:** User-facing behavior and success criteria.
    - **HOW:** Implementation details, files, and logic.
    - **BUILD READINESS:** Dependencies, Risk Level (Low/Med/High), and Parallel-safety.
- **Handoff:** Branch name is added to the PM task notes $\rightarrow$ Task tagged `status-ready-to-build`.

### 3. Build & Implementation (The "Execution")
- **BuildProcessCoordinator Pickup:** BuildProcessCoordinator reads the branch name $\rightarrow$ creates a git worktree $\rightarrow$ dispatches a Coder agent.
- **Coder Agent:** Implements the feature on the feature branch.
- **Verification:** Coder updates `STATE.md` and `PLAN.md` $\rightarrow$ tags `status-ready-for-qa`.

### 4. QA & Review (The "Quality Gate")
- **Functional QA:** A dedicated QA agent verifies the acceptance criteria and runs tests.
- **UX QA:** A **UX QA Agent** validates the interaction flow and "feel" against the UX Design Doc.
- **Specialized Reviews:** Three parallel review passes (Security, Performance, User Value) check for non-functional requirements.
- **Rework Loop:** If any gate fails, feedback is written to a file $\rightarrow$ Coder agent fixes $\rightarrow$ QA/Review re-runs.

### 5. Documentation & Shipping (The "Closure")
- **Docs Gate:** The Site Agent generates a user-facing doc page. The feature cannot ship without a URL.
- **Merge:** BuildProcessCoordinator merges the feature branch to main.
- **Shipping:** Task marked `status-shipped` → Project version bumped → Release notes generated → DMG/Binary published → DMG dropped to `~/Downloads/hermes/hos/` for testing access.
- **Multi-Platform TestFlight (MANDATORY):** EVERY release ships ALL platforms — not just Mac:
  - Mac Server DMG (notarized + published via `package-release.sh` + `publish-release.sh`)
  - iPhone Companion (archive `hOS` scheme → TestFlight)
  - iPad "hOS Shared View" (archive `hOSiPad` scheme → TestFlight)
  - Watch app (embedded in iPhone Companion, no separate upload)
  - Release is NOT complete until all TestFlight builds reach VALID state.
- **Human Testing (The "Final Seal"):** A dedicated **Human Testing Agent** (`human-testing.md`) guides the product owner through structured release testing on a real machine. It reads feature doc pages to know what "working as intended" looks like, walks the tester step-by-step, and makes the call on each result:
  - ✅ **PASS** — matches docs $\rightarrow$ log and continue
  - 🐛 **BUG** — doesn't match docs $\rightarrow$ create Asana bug task (`status-blocked`), keep moving, **never fix in-session**
  - 💡 **NEW FEATURE** — works as specced but user wants more $\rightarrow$ create Asana task (`status-ready-to-plan`), continue testing
  - Session ends with a summary: pass/bug/feature counts + ship/no-ship recommendation.
  - Only after this manual sign-off is the release considered "Stable".

## Agent Role Matrix
| Role | Primary Responsibility | Guardrail |
|---|---|---|
| **Build Manager** (cron, autonomous) | Triage, Dispatch, Merge, Release | **NEVER** writes source code; **NEVER** commits directly to main |
| **Process Coordinator** (interactive, Piyush-driven) | Oversight, status, process improvement | **NEVER** edits Swift/source; **NEVER** runs xcodebuild; **NEVER** pushes to main |
| **Requirements** | Research, Spec writing, Branching | Focuses on "What" and "How" (not implementation) |
| **UX Designer** | User Journeys, Layouts, Micro-copy | Focuses on interaction intent, not implementation |
| **Coder** | Implementation, Unit Tests | Works only on assigned feature branches; cannot merge PRs |
| **QA** | Spec verification, Integration testing | Cannot edit code; must independently verify audits (not accept coder self-reports) |
| **UX QA** | Usability, Friction, Consistency | Ignores bugs; focuses on the "feel" and flow |
| **Reviewer** | Security, Perf, UX auditing | Focuses on edge cases and quality, not functionality |
| **Site Agent** | Technical & Marketing docs | Ensures no "docs drift" between code and site |
| **Human Testing** | Guides product owner through release testing on real device | **NEVER** fixes bugs; logs them as Asana tasks and keeps moving |

### Role Boundary Violations (Known Failures — Do Not Repeat)
- **Process Coordinator editing source directly:** Caused Mac Server regression (Postgres startup failure). Required DMG rollback. Aug 2026.
- **QA accepting coder self-declared audits:** Coder said "@Environment audit complete"; QA didn't verify independently. v0.6.11 shipped with same crash class. Aug 2026.
- **QA approving guards without runtime verification:** SafeCKContainer guard read Info.plist for entitlements (impossible — they're in codesign signature). Always returned false. Silently broke CloudKit sync across all versions. Aug 2026.
- **CURRENT_PROJECT_VERSION set to semver string instead of integer:** ASC requires an integer build number (94, 95…). Setting it to "0.6.13" causes ASC to reject uploads silently. MARKETING_VERSION = semver string; CURRENT_PROJECT_VERSION = integer only. Verify before every release: `grep CURRENT_PROJECT_VERSION project.pbxproj` — must be a plain integer.
- **MISSING_EXPORT_COMPLIANCE blocking TestFlight:** Builds get stuck and never reach VALID until manually patched via ASC API. Permanent fix: add `ITSAppUsesNonExemptEncryption = NO` (boolean false) to all target Info.plist files. Must be in ALL targets (Mac + iPhone + iPad). No code change needed — plist key only. Do this once at project setup and never again.
- **Coordinator bypassing package-release.sh for DMG creation:** Running ad-hoc `hdiutil create` directly on the .app produces a DMG missing the /Applications symlink and with the wrong volume name — unusable for drag-to-install. The ONLY valid way to produce a DMG is `package-release.sh`. No exceptions. Verify every DMG by mounting it: must show the app + Applications symlink, volume name "hOS Server", and correct CFBundleVersion integer.
- **Never set `workdir` on long-running cron jobs:** Hermes holds a filesystem lock on the `workdir` for the entire duration of the job. A 20-minute requirements agent run will block every other cron that overlaps. Let agents `cd` themselves inside their prompt instead — same behavior, no lock contention.

## QA Mandatory Additions (Aug 2026)

### Runtime Correctness of Guards
For any new guard or validation method: QA must identify the data source, verify it actually contains the expected key at runtime, and explain WHY the guard returns the correct value. Structural presence ≠ behavioral correctness.

### Independent Audit Verification
When a task claims an exhaustive audit ("X was the only instance"): QA must independently re-run the same search and confirm the count. Do not accept coder's self-reported count.

## iOS Device Confirmation Gate (Mandatory)

`processingState=VALID` in ASC does NOT mean releasable. Required before `status-released` on any iOS task:
1. App VALID in ASC ✅
2. Piyush installs on real device ✅
3. App launches without crashing 10+ seconds ✅
4. Piyush explicitly confirms ✅

Simulators don't count — CloudKit crashes reproduce only on real devices with real iCloud accounts.

## Communication & Session Model

### Dual-Session Pattern
The product owner interacts with the coordinator agent through **two equal surfaces**:

| Surface | Usage | Context |
|---|---|---|
| **GUI (Desktop App)** | Deep work, file inspection, visual tasks | Same context files |
| **Slack (#piyush-mm4p-hosbuildprocess)** | Quick commands, status checks, remote control | Same context files |

Both sessions share the same agent context files, PM project, and git repos.
A command from Slack and a command from the GUI have equal authority. The
coordinator agent reads the same context files regardless of which surface
the command originated from.

### Communication Rules
- **Delta-only reporting:** Agents never report "no changes" or "everything is fine." Silence = healthy pipeline.
- **Asana notes are the contract:** If it's not in Asana task notes, it didn't happen.
- **Slack commands from Piyush** are equivalent to GUI chat commands.

## Shared Agent Context
ALL agents read `SHARED-CONTEXT.md` at session start before their role-specific
file. It contains cross-cutting state that affects every role:

- Project identity & current version
- S-S-D model (what lives where)
- Communication channels (Slack, Telegram, Asana, COORDINATION.md)
- Repo layout (4 checkouts, stay in your lane)
- Asana tag system (status, phase, agent tags with GIDs)
- Tool & model matrix (which CLI, which model, which escalation path)
- Release pipeline (full flow from ready-to-build → released)
- Concurrency guardrails (baton pattern, lock files, lease-based statuses)
- Cross-role boundary table (what each role cannot do)
- iOS device confirmation gate
- Known issues & technical debt

**Individual role files** (`<role>.md`) build on top of the shared context.
If a conflict exists, the role file wins for role-specific behavior; the
shared file wins for shared state and process rules.

**Note on agent context files:** `docs/agent-context/*.md` are runtime files
living in the requirements branch — they are NOT synced to PiyushBuildOS.
PiyushBuildOS contains process framework (PROCESS_FRAMEWORK.md) and role
stubs only. Runtime context evolves per-project; the framework is reusable.

## Key Process Artifacts
- **Change Checklist:** The master contract for the process.
- **Decisions Log:** A record of all "blocking" architectural decisions to avoid re-litigating.
- **Agent Context Files:** Specialized instructions (system prompts) for each role.
- **Shared Context File:** Cross-cutting state read by ALL agents at session start.
- **Build Readiness Metadata:** Essential data (Risk/Scope) used by the BuildProcessCoordinator to serialize builds.
