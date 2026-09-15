# HANDOFF — PiyushBuildOS (process repo)

_Updated 2026-09-15. This repo owns the BUILD PROCESS, not the product.
Product code: `hos-monorepo` (see its HANDOFF.md). Runtime state:
`~/code/hos-state/` (NOT git — copy manually when migrating machines)._

## Where we are

- **main @ `6b8c099`** — pushed clean. Recent commits: task-mode charter
  (`9c19a45`), watchdog role updates (`6b8c099`: `[SILENT]` verbatim rule +
  detached-dispatch lesson).
- **v0.7.0 hardening phases A/A2/B/C complete** (branch cleanup ~350→61,
  three-layer separation, STATUS.json schema 2, model pins + gate
  contracts, oversight-charter.md). Phase D executed end-to-end: D0–D3
  built, merged (PRs #40–#43), tagged shipped.
- **v0.7.0 release gate**: blocked ONLY on human device tests of
  D2 (approval round-trip) + D3 (chat→LLM) — currently waiting on Piyush
  installing TestFlight b101 on iPad (contains the CK-identity hotfix,
  branch `feature/ck-identity-no-lan` in hos-monorepo).

## Open process items

1. **Dispatcher-role gap** — cron `e4b0b407e0fb` runs
   `roles/monitor-watchdog.md` (thin observer) while
   `roles/build-manager.md` claims the same cron as dispatcher. Option A
   recommended (dispatch-first with inline watchdog checks); Piyush never
   decided. D1–D3 shipped via manual-dispatch precedent — fix before the
   next multi-task build wave.
2. **Watchdog skips STATUS.json step-5 updates** — minor, same fix batch.
3. **mm4p not registered** in ADTools deploy-signals machine registry
   (`state/deploy-signals/machine-signals.json`) — Piyush's call.
4. **Event-driven pipeline** (stage-completion triggers next instead of
   cron-poll) — deferred until pipeline stable; Piyush wants it eventually.

## Lessons learned (process-level)

- **Role files win over cron prompts** (model-pin decision #39). When a
  cron's prompt and its role file disagree, the agent flails silently —
  one owner per cron, stated in exactly one file.
- **Mechanical alerts, never delegated**: trigger.sh itself posts Slack
  per-transition (🚧 advance, ⏳ queue, 🚨 fail, ✅ release complete).
  Agents must never be responsible for stage alerts.
- **Coders dispatched from cron sessions die when the turn ends** —
  dispatch detached (nohup … &), verify ~/.claude/projects/<worktree>/
  <session>.jsonl exists before reporting the dispatch.
- **`[SILENT]` must be output verbatim** (six characters) — a cron
  translating it caused a degenerate loop that hallucinated files.
- **Hermes-level workdir locks are per-checkout, not per-branch** —
  stagger cron schedules across machines sharing a repo; never remove the
  workdir key.
- **STATUS.json (schema 2) in hos-state is the single status point** —
  every actor logs before responding; claims must cite a file.
- **Session task mode** (charter, agreed Sep 12): every actionable request
  = tracked task (todo + STATUS.json + Asana); heavy/isolatable units
  dispatch to subagents; oversight/decisions stay inline.
- **3-LAYER RULE**: A = hos-monorepo (product), B = this repo (process),
  C = hos-state (runtime, never git). Never mix.
- **ADTools is a deploy target** — edit source at
  `/Users/Shared/codebase/projects/ADTools`, then sync to `~/ADTools`.
- **Branch hygiene**: delete-on-merge with tip-SHA logged; `-fresh`/`-v2`/
  `-rebased` naming banned.

## Machine-specific bits (move with the project)

- Cron jobs live in Hermes config on the build Mac (coordinator
  `e4b0b407e0fb` 10m → Slack `C0BRKHLDB7Z`; requirements `d883a2abf8d5`
  20m :05/:25/:45 → `C0BRUMJ0743`; drift audit paused; state backup daily
  3am). Recreate via Hermes cron tooling, not launchd.
- Slack channels: build `C0BRKHLDB7Z`, requirements `C0BRUMJ0743`,
  general `C0BS2SRJ8RF`, dashboard `C0BRZ0T6HUN`, drift `C0BS4JZ700L`,
  fleet `C0BRKHL3M2T`.
- Asana project `1217507880139390`; tag GIDs in hos-requirements HANDOFF.md.
- Claude CLI dispatches must `source ~/ADTools/config/secrets.conf` —
  never hardcode env vars.
