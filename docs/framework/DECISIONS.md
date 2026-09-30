# Decision Points — Unified SDLC Framework

**Status:** OPEN — Piyush rules on each; answers recorded inline with date.
**Companion doc:** `PROPOSED_FRAMEWORK.md`. Crosswalk evidence: `ANALYSIS.md`.

Format: one decision at a time may be answered via chat/Slack; record answer + date here.

---

## D-U1. Merge authority — human always, or auto-merge on small-change lane only?

Playbook A says an AI review never counts as approval; B currently auto-merges on AI security
pass (no human gate). Proposal: named human approves every merge; auto-merge survives only on
the small-change lane (typo/dep/config) with pre-delegated approval.

**Options:**
- (a) Human approves all merges (strictest, matches A)
- (b) Human approves standard/high lanes; auto-merge on small-change lane only (proposed)
- (c) Keep current auto-merge (B as-is)

**Decision:** _(pending)_

---

## D-U2. Intent-layer tooling — migrate off Asana free tier?

Unified model requires: hierarchy, PR-reviewed requirement docs in repo, CODEOWNERS routing,
traceability automation. Asana free tier = tags + sections only, notes-as-contract must be retired
either way.

**Options:**
- (a) GitHub Issues + Projects + sub-issues (native to the repo, traceability CI-friendly) — proposed
- (b) Jira (if a client/enterprise constraint appears)
- (c) Keep Asana, add repo-based docs layer (weakest: two SSOTs for status)

**Decision:** _(pending)_

---

## D-U3. Branch budget — is ≤ 2–3 days lifetime + merge queue acceptable for hOS?

hOS releases are multi-platform (DMG + 2× TestFlight) with CloudKit single-writer constraints.
Trunk-based + feature flags changes release mechanics.

**Options:**
- (a) Adopt for all repos now (hOS pilot)
- (b) Adopt for new projects first; hOS moves at v0.8.0
- (c) Adopt everywhere except release-critical repos

**Decision:** _(pending)_

---

## D-U4. Sandbox farm hosting — which machine(s)?

Sandbox farm (ephemeral per-task containers) + state tier (Postgres/MinIO/Redis) need a home on
the mesh. Candidates: dedicated LXC on existing infra; laptop-m1; mm4p (already canonical build
machine — prefer NOT to also run state tier there; disk incidents argue for separation).

**Options:**
- (a) Dedicated LXC node for sandbox farm + state tier (proposed)
- (b) laptop-m1 hosts state tier; sandbox farm on LXC
- (c) All on mm4p (rejected in analysis: disk-full history + single-writer conflicts)

**Decision:** _(pending)_

---

## D-U5. Orchestrator engine — when (if ever) to move off trigger.sh?

Current shell envelopes implement the manager-worker pattern and are incident-proven. LangGraph/
Dify adds graph complexity + a new runtime to govern.

**Options:**
- (a) Stay on trigger.sh until parallel epics/fan-out reviewers outgrow it, then ADR (proposed)
- (b) Adopt LangGraph now (state tier makes persistence easier)
- (c) Adopt Dify visual platform (lower bar for non-engineering contributors)

**Decision:** _(pending)_

---

## D-U6. Requirement ID scope — hOS backfill or forward-only?

~110 shipped features have no FR-IDs. Backfilling is real work; forward-only leaves old features
untraceable.

**Options:**
- (a) Forward-only from v0.8.0 (proposed)
- (b) Backfill v0.7.x features as they're touched
- (c) Full backfill before unified model is "active"

**Decision:** _(pending)_

---

## D-U7. Governance charter for hOS — who holds which hat today?

Charter needs names now, even if all hats are Piyush. Which hats (if any) go to agents-as-
delegates-with-human-signoff vs stay purely human?

**Options:**
- (a) Piyush holds all 5 hats initially; revisit at first team hire (proposed)
- (b) Delegate QA-owner verification to QA agent with Piyush sign-off
- (c) Delegate architecture-owner prep to coordinator with Piyush ADR approval

**Decision:** _(pending)_

---

## D-U8. Data classification — handle the PII-in-repo issue before or with U1?

Analysis flagged ~112 PII occurrences in requirements material. A's playbook requires
classification + redaction before AI sees sources.

**Options:**
- (a) Redaction pass bundled into Phase U1 (proposed — it's the governance layer's first test)
- (b) Separate incident-driven fix first, U1 after
- (c) Defer to U4 (governance ops)

**Decision:** _(pending)_
