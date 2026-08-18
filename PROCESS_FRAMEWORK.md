# PiyushBuildOS — Agentic Build Process Framework

This folder contains the consolidated process, roles, and context for the "Piyush-style" agentic build-out of complex software systems. This is a general-purpose framework derived from the hOS project, designed to be reused for new projects.

## Core Philosophy: The "S-S-D" Model (Single-Sourced Data)
The fundamental rule of this process is that every piece of information has exactly one source of truth to prevent drift and hallucination.
- **Product Intent (The "What"):** Managed in a Project Management tool (e.g., Asana). This is the a-priori truth for requirements, specs, and acceptance criteria.
- **Build State (The "How/Status"):** Managed in Git. Branch names, commit history, and state files (`STATE.md`, `PLAN.md`) are the evidence of progress.
- **User-Facing Truth:** Managed via a dedicated Marketing/Docs site. No feature is "Shipped" until its documentation exists.

## The 4-Checkout Layout (Role Isolation)
To prevent "context pollution" and accidental edits, different agents work in isolated checkouts:
1. **Monorepo/Coordinator:** (Main branch) Only the Coordinator agent commits here. Handles merges, worktrees, and final releases.
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
- **Coordinator Pickup:** Coordinator reads the branch name $\rightarrow$ creates a git worktree $\rightarrow$ dispatches a Coder agent.
- **Coder Agent:** Implements the feature on the feature branch.
- **Verification:** Coder updates `STATE.md` and `PLAN.md` $\rightarrow$ tags `status-ready-for-qa`.

### 4. QA & Review (The "Quality Gate")
- **Functional QA:** A dedicated QA agent verifies the acceptance criteria and runs tests.
- **UX QA:** A **UX QA Agent** validates the interaction flow and "feel" against the UX Design Doc.
- **Specialized Reviews:** Three parallel review passes (Security, Performance, User Value) check for non-functional requirements.
- **Rework Loop:** If any gate fails, feedback is written to a file $\rightarrow$ Coder agent fixes $\rightarrow$ QA/Review re-runs.

### 5. Documentation & Shipping (The "Closure")
- **Docs Gate:** The Site Agent generates a user-facing doc page. The feature cannot ship without a URL.
- **Merge:** Coordinator merges the feature branch to main.
- **Shipping:** Task marked `status-shipped` $\rightarrow$ Project version bumped $\rightarrow$ Release notes generated $\rightarrow$ DMG/Binary published.
- **Human Validation (The "Final Seal"):** After the binary is published (`status-released`), a formal **Human QA Plan** (derived from the Must-Beta feature list) is executed on a reference machine. This phase validates the "Clean Install" experience and end-to-end vertical slices on real household data. Only after this manual sign-off is the beta considered "Stable".

## Agent Role Matrix
| Role | Primary Responsibility | Guardrail |
|---|---|---|
| **Coordinator** | Triage, Dispatch, Merge, Release | **NEVER** writes source code. |
| **Requirements** | Research, Spec writing, Branching | Focuses on "What" and "How" (not implementation). |
| **UX Designer** | User Journeys, Layouts, Micro-copy | Focuses on interaction intent, not implementation. |
| **Coder** | Implementation, Unit Tests | Works only on assigned feature branches. |
| **QA** | Spec verification, Integration testing | Cannot edit code; reports PASS/FAIL. |
| **UX QA** | Usability, Friction, Consistency | Ignores bugs; focuses on the "feel" and flow. |
| **Reviewer** | Security, Perf, UX auditing | Focuses on edge cases and quality, not functionality. |
| **Site Agent** | Technical & Marketing docs | Ensures no "docs drift" between code and site. |

## Key Process Artifacts
- **Change Checklist:** The master contract for the process.
- **Decisions Log:** A record of all "blocking" architectural decisions to avoid re-litigating.
- **Agent Context Files:** Specialized instructions (system prompts) for each role.
- **Build Readiness Metadata:** Essential data (Risk/Scope) used by the Coordinator to serialize builds.
