# Unified SDLC Framework — Convergence Proposal

**Status:** PROPOSED (v0.1 draft — under review by Piyush)
**Source analysis:** full crosswalk at `ANALYSIS.md` in this directory; framework comparison vs
`AI-Assisted_SDLC_WaysofWorking_BestPractices_1.md` (Accelerating Digital playbook, v1.1).
**Visuals:** `framework-visuals.html` (deployment topology + requirements flow — open in a browser).
**Related:** `DECISIONS.md` in this directory (open decision points Piyush must rule on).

---

## Why this exists

Two SDLC frameworks are in play:

- **Framework A** — "AI-Assisted SDLC: Ways of Working" (team playbook, top-down, GitHub-native,
  human ceremonies, 10 phases). Strong on: requirement IDs, docs-as-code, CI quality gates,
  human approval model, request ledger, AI data governance.
- **Framework B** — PiyushBuildOS (this repo), built bottom-up from 15 logged process incidents
  while shipping hOS solo-with-agents. Strong on: machine-checkable gate contracts, incident
  corpus with gap-class taxonomy, role boundaries with receipts, human-testing/device-confirmation
  release gates, event-driven pipeline, delta-only reporting.

**Verdict of the crosswalk: converge, don't merge.** B is the better starting point (its governance
layer is team-ready); A is the better intent layer (IDs, CI gates, approvals). As-is adoption:
A onto solo+agents 2/5, B onto a team 2/5, unified model 4/5. This document defines that unified
model so one framework can run hOS today and a multi-person team tomorrow.

**The three gaps neither framework covers alone** (drove this proposal):
1. **Parallel work at team scale** without repo/worktree explosions (B's serialized pipeline
   idles a team; A's ceremonies don't fix conflicts).
2. **Architecture coherence** — requirements must consider shared services and system-wide impact.
3. **Governance** — named human approvers per area (PO, UX, architecture, security, QA) assigned
   at project start, so "who approves what" is never ambiguous.

---

## Part 1 — The unified model (stage by stage)

One loop: **requests → requirements → spec → backlog → build → review → test → release → operate**,
with artifacts versioned in the repo and status tracked in the tracker. Human gates are named
people; agents execute everything between gates.

| Stage | Artifact (repo) | Tracker (Jira/Asana) | Agent role | Human gate |
|---|---|---|---|---|
| Intake | `docs/requirements/request-ledger.md` — every stakeholder request gets REQ-ID | feature-request card | drafting from chat/meetings | **PO**: accept / park / reject — every REQ resolved, none blank |
| Requirements | `docs/requirements/{BR,FR,NFR}-xxx.md` — permanent IDs, Gherkin ACs, **SYSTEM IMPACT** section | epic + stories | AI drafts + critiques; UX design doc for UI features | **PO + Architect** approve requirements PR |
| Spec | `docs/specs/TSD-xxx.md` + `docs/adr/ADR-xxxx-*.md` (per-decision files) | — | option analysis, checklist prep | **Architect** approves; architecture checklist required for High-risk features |
| Backlog | `backlog/TSD-xxx.yaml` | cards w/ estimates, deps, `ai-suitable` | generation + sync | **PO** approves scope |
| Build | feature branch `feat/<req-id>-<slug>` | card → In Progress | coder agent in **ephemeral sandbox** | none (agent-executed) |
| Review | PR with evidence attached | card → In Review | AI review fleet (security/perf/user-value) — **inputs with citations, never approvals** | **Area owner(s)** approve merge (CODEOWNERS-routed; 2 approvals for security-relevant) |
| Test | test suites + `docs/traceability.md` (CI-regenerated) | — | QA agent exploratory + independent audit verification | **QA owner** signs release readiness |
| Release | release notes, risk summary | card → Shipped | packaging pipeline (canonical tool only) | **PO** go/no-go + device confirmation |
| Operate | postmortems, metrics | bug / feature cards | watchdog, drift-audit, feedback drafting | PO spot-checks feedback summaries |

**Load-bearing rules inherited from B (non-negotiable):**
- Gate contracts: every stage transition has machine-checkable required inputs; failure →
  `status-blocked` + Slack alert. No soft passes.
- Role boundaries with incident receipts (oversight charter); state claims require a cited file
  path; session memory is cache, state tier is truth; write-back rule before turn end.
- Release: canonical packaging tool only, mount-verify, smoke test, multi-platform (DMG +
  TestFlight), device-confirmation gate before released.

**Adopted from A (closes B's deficits):**
- Permanent requirement IDs + PR-reviewed requirement docs (retires "task notes are the contract" —
  tracker holds status + links, never content).
- Real CI on every PR: 0 failing tests, coverage on changed lines, scans. The QA agent shifts from
  being-the-CI to exploratory + independent verification.
- Request ledger, blameless product postmortems, DORA-lite + escaped-defect metrics.
- Human Approve on merge and release (retires auto-merge except small-change lane).
- AI data governance: data classification, approved-tools/MCP allowlist as recorded decision,
  quarterly access review.

**Dropped from both (ceremonial):** sprints, standups, backlog-YAML sync mechanics, pairing rules,
400-line PR limits, estimate/capacity apparatus, dual-session parity mechanics. Event-driven
pipeline + delta-only reporting remain the operating mode at every team size.

---

## Part 2 — Parallel work without repo explosions

Root cause of past failures: long-lived branches + multiple standing checkouts + per-worktree state.
Fix = change *what lives in parallel*:

| Mechanism | Replaces | Rule |
|---|---|---|
| Trunk-based + feature flags | long-lived feature branches | code merges early behind a flag; incompleteness lives in flags |
| Short branch budget | weeks-old `feature/*` branches | branch lifetime ≤ 2–3 days, one PR-sized unit; stale > 3 days = drift-audit alert |
| Merge queue | serialized one-task pipeline | parallel work; only final integration test serializes |
| One checkout per human; sandboxes for agents | 4-checkout layout | worktrees become ephemeral per-task containers (sandbox farm), destroyed on merge |
| CI on every PR | QA-agent-as-CI | conflicts + test failures surface in minutes |
| Centralized packaging | ad-hoc local builds | only the canonical build machine produces releases; humans never build |

---

## Part 3 — Architecture coherence (system impact)

- **System Context Map** (`docs/architecture/`): C4 container diagram + **shared-services
  registry** (each shared service with a named owner). Drift-audited like all docs.
- **Spec template gains SYSTEM IMPACT** (4th section alongside WHAT/HOW/BUILD READINESS):
  containers touched, shared services touched, contracts changed (CK record shapes, API, schema),
  migrations, who else is affected. Blank SYSTEM IMPACT = QA/reviewer rejection.
- **ADR trigger rule:** full ADR when a decision crosses a container boundary, touches a shared
  service/contract, adds a dependency, or changes the data model; otherwise a one-line note.
- **Shared-code change rule:** changes to a registered shared service are their own task,
  approved by that service's owner. A feature can never silently widen into shared code
  (path-based tripwire via CODEOWNERS).

---

## Part 4 — Governance charter (named humans, per project)

One page, committed at project kickoff. Small teams: one person, several hats — approvals stay
separate events.

| Gate | Approver | Enforced by |
|---|---|---|
| Requirements accepted / prioritized | Product Owner | CODEOWNERS on `docs/requirements/` |
| Spec + ADRs | Architecture owner | CODEOWNERS on `docs/adr/`, `docs/architecture/` |
| UX flows / designs | UX owner | design doc approval before story DoR |
| Security-relevant PRs, scan exceptions | Security owner | CODEOWNERS on auth/PII/entitlements paths; 2-approver rule |
| Test / release readiness | QA owner | traceability report + gate sign-off |
| Merge | PO or delegate — **a human, always** | branch protection: no auto-merge on standard/high lanes |
| Ship + device confirm | PO | device-confirmation gate |

Escalation: cross-area conflicts → PO. Auto-merge survives **only** on the small-change lane
(typo, dep bump, config) where an area owner has pre-delegated.

---

## Part 5 — Deployment topology

See `framework-visuals.html`, diagram 1. Summary:

**Per-person (private):** local checkout, optional sandboxed local agent daemon, oversight surface
(GUI/Slack). Nothing stateful.

**Shared (run once, on the Tailscale-mesh LXC/Docker hosts):**
- **Orchestration node** — manager-worker orchestrator (today: proven trigger.sh envelopes;
  LangGraph/Dify only as a later ADR), tracker ingestion (gate-contract validated, not just a poll),
  AI review fleet, requirements + human-testing agents.
- **Sandbox farm** — ephemeral per-task Docker containers (checkout + coder + tests), destroyed on
  merge. Replaces shared worktrees; conflict surface shrinks to the merge queue.
- **State tier** — Layer C as a service: Postgres (STATUS, pipeline-stats, gate results, leases),
  object store (test artifacts, QA evidence, feedback files), Redis (queues, locks, heartbeats).
  One-writer-per-stage rules carry over.
- **Model tier** — existing LiteLLM GPU node; per-role model matrix + MODEL PINs, per-sprint token
  spend tracking.
- **Release pipeline** — one canonical build machine (mm4p-class); package tool is the only path;
  artifact verification mandatory.

**Corrections over the generic orchestrator pattern** (from incident corpus): the PR must carry
traceability evidence, not just a branch; release packaging never runs in a sandbox or dev node;
connectivity independence holds only because state lives in the state tier, not agent sessions.

**Sequencing:** (1) state tier + sandbox containers replace worktrees — biggest parallelism win,
no new engine; (2) tracker ingestion + PR post-back — team collaboration surface; (3) LangGraph/
Dify — later ADR if graph complexity outgrows shell orchestration.

---

## Migration path for this repo

The convergence lands in PiyushBuildOS in phases, each independently useful:

1. **Phase U1 — intent layer:** requirement IDs + request ledger + spec PRs in product repos;
   governance charter template (`PROJECT_GOVERNANCE.template.md`).
2. **Phase U2 — quality layer:** real CI + coverage gates on product PRs; QA agent role narrowed
   to exploratory + independent verification.
3. **Phase U3 — scale layer:** state tier services + sandbox farm; merge queue; branch budget +
   drift-audit rule updates.
4. **Phase U4 — governance ops:** CODEOWNERS routing, governance charters per project, metrics +
   postmortem practice, AI data-governance ADR.

Each phase ships as its own PR set against product repos + updates here. hOS is the pilot;
new projects start directly at the unified model.
