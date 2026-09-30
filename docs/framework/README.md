# Unifying the SDLC Frameworks — Discussion Guide

**Audience:** collaborators joining the PiyushBuildOS process work. Read this before commenting
on the framework PRs.
**Status:** PROPOSED (v0.1) — open for discussion via PR comments on this branch.
**Companion docs:** [`PROPOSED_FRAMEWORK.md`](PROPOSED_FRAMEWORK.md) (the proposal) ·
[`DECISIONS.md`](DECISIONS.md) (decision points awaiting rulings) ·
[`ANALYSIS.md`](ANALYSIS.md) (full crosswalk evidence) ·
[`framework-visuals.html`](framework-visuals.html) (deployment topology + requirements flow diagrams).

---

## TL;DR

Two frameworks are being converged into one:

| | Framework A — "AI-Assisted SDLC" playbook | Framework B — PiyushBuildOS (this repo) |
|---|---|---|
| Origin | Top-down team playbook (Accelerating Digital, v1.1) | Bottom-up, forged by 15 logged process incidents while shipping hOS solo-with-agents |
| Strengths | Requirement IDs, docs-as-code, CI quality gates, human approval model, request ledger, AI data governance | Machine-checkable gate contracts, incident corpus, role boundaries with receipts, human-testing/device-confirmation release gates, event-driven pipeline |
| Weakness | Half is people-shaped ceremony unusable without a team; no artifact-level release gates | No requirement IDs, no real CI/tests, auto-merge (no human approval), solo-shaped tooling |

**The unified model takes B's governance + release machinery as the skeleton and A's intent +
quality layers as the flesh.** Full stage-by-stage mapping is in `PROPOSED_FRAMEWORK.md` Part 1.

## The three problems that shaped it

1. **Parallel work without repo explosions.** Past failures came from long-lived branches,
   multiple standing checkouts, and shared worktree state. The fix is structural: trunk-based
   development with feature flags, ≤2–3-day branch lifetimes, a merge queue, and ephemeral
   per-task sandbox containers for agent work instead of shared worktrees.
2. **Architecture coherence.** Every spec gains a **SYSTEM IMPACT** section (shared services
   touched, contracts changed, who else is affected). Shared services get a registry with named
   owners; changes to them are their own task approved by that owner. Big decisions become
   per-decision ADRs.
3. **Governance.** At project kickoff, a one-page charter names the human approvers: Product
   Owner, Architecture, Security, UX, QA. CODEOWNERS routes PRs to them. AI review produces
   cited evidence but never an approval. One human may hold several hats; approvals stay
   separate events.

## What runs where (see `framework-visuals.html`)

- **Per person:** local checkout, optional sandboxed agent daemon, oversight surface. Nothing stateful.
- **Shared:** orchestration node (tracker ingestion, gate validation, agent dispatch, review fleet),
  sandbox farm (ephemeral containers), state tier (Postgres/MinIO/Redis — Layer C as a service),
  LiteLLM model tier, and one canonical release machine.

## How to contribute

1. Read `PROPOSED_FRAMEWORK.md` end to end (it's short — five parts).
2. Look at the diagrams in `framework-visuals.html` (open in a browser).
3. Read `ANALYSIS.md` if you want the full evidence trail — every verdict is cited to a section
   of one of the two source frameworks.
4. Comment on this PR with: **[PART-1..5]** prefix for proposal content, **[D-U#]** prefix for
   decision points, **[GAP]** for anything both frameworks miss.
5. Rulings on `DECISIONS.md` are made by Piyush; answers get recorded inline with dates.

## Non-negotiables (inherited from incident history)

These survived every framework comparison and are not open for debate in this round:

- No agent or human pushes directly to main; everything merges through a reviewed PR.
- AI review output is evidence with citations — it never substitutes a human approval.
- Releases are produced only by the canonical packaging pipeline and verified as artifacts
  (mount, smoke test, all platforms VALID, device confirmation).
- Gate transitions are machine-validated; a failed gate hard-blocks with an alert, never soft-passes.
- Runtime state lives in the state tier, not in agent sessions or chat history. State claims
  require a cited source.

## Phasing

U1 intent layer (IDs, ledger, governance charter template) → U2 quality layer (CI + coverage) →
U3 scale layer (state tier, sandboxes, merge queue) → U4 governance ops (CODEOWNERS, charters,
metrics, postmortems, data governance). hOS pilots; new projects start directly at the unified
model. Detail in `PROPOSED_FRAMEWORK.md` § Migration path.
