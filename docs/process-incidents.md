# Process Incident Corpus — hOS Build Pipeline

Purpose: every known process failure, its root cause, and the process-gap class it belongs to.
New incidents get appended with date + class. Drift audit reviews this file monthly for recurring classes.

| # | Date | Incident | Root cause | Gap class | Fixed by |
|---|------|----------|-----------|-----------|----------|
| 1 | 2026-08-22/23/30 | Coordinator edited source directly (3 corrections from Piyush) | Role boundary not codified — charter missing | Role ambiguity | hOS ROLE BOUNDARY rules + oversight charter (Phase C) |
| 2 | 2026-08-30 | Raw xcodebuild+cp to /Applications broke signing → Mac Server regression, rolled back to v0.6.10 | Bypassed package-release.sh | Tool bypass | Release checklist mandatory (build-manager.md) |
| 3 | 2026-08-30 | 4 failed ASC API attempts (assign builds to beta groups) | Guessing API shapes without research | External-system discipline | "Read-only first, max 1 mutating call per unknown endpoint" guardrail |
| 4 | 2026-09-04 | Requirements agent + build manager workdir lock contention (5460s timeout) | Both crons had workdir on same checkout; lock is Hermes-level | Concurrency | Staggered schedules; never remove workdir (AGENTS.md injection depends on it) |
| 5 | 2026-09-04 | Coordinator cron delivered to wrong Slack channel (C0BS20BT1GA) | Deliver target misconfigured | Config drift | cron deliver fixed to slack:C0BRKHLDB7Z |
| 6 | 2026-09-05 | LiteLLM proxy down → coder 401 "key invalid", coder stuck, zero progress | External dependency outage without health gate | Dependency health | Watchdog detected; add provider health check before dispatch |
| 7 | 2026-09-05/06 | exportArchive "Failed to Use Accounts" (plist apiKey not honored) | Non-interactive ASC auth requires CLI flags | Tool knowledge | -authenticationKeyPath/-KeyID/-IssuerID flags (documented in build-manager.md) |
| 8 | 2026-09-06 | b94 DMG missing Applications symlink + wrong volume name | Coordinator ran ad-hoc hdiutil instead of package-release.sh | Tool bypass (same class as #2) | DMG verification steps in process docs; pitfall entry |
| 9 | 2026-09-06 | b95 Mac DMG had CFBundleVersion=0.6.13 despite iOS=95 | package-release.sh `agvtool new-version -all` stamps MARKETING version into build number + dirty tree built | Script defect + dirty-tree gap | agvtool fix queued as coder task; pre-flight clean-tree check |
| 10 | 2026-09-06 | /tmp/hos-build-cloudkit-sync-panel-test-approval was a SYMLINK to ~/code/hos-monorepo | Coder agent created fake worktree | Worktree hygiene | PiyushBuildOS rule: readlink check before packaging; never symlink worktrees |
| 11 | 2026-09-09 | Build manager cron degenerate loop 23:55→04:54 (hallucinated watchdog-state.md; real file .json; zero work done) | LLM reconstructed state from memory instead of reading the file; wrong filename in role doc | State hallucination | monitor-watchdog.md file ref corrected; anti-hallucination rule (state claims must cite files) in charter |
| 12 | 2026-09-09 | Manual cronjob run no-op while scheduler tick executing (execution_skipped) | Manual run collides with scheduled run | Concurrency | Documented; stagger manual runs |
| 13 | 2026-09-12 | Runtime/process state (watchdog-state.json, pipeline-stats, COORDINATION.md, req-agent state) git-tracked in BOTH product checkouts, drifting against each other; role docs in 3 places | Layers mixed: product repo held process + runtime state | Layer separation (structural) | Three-layer migration 2026-09-12: Layer C = ~/code/hos-state/ (not git), role docs single-copy in PiyushBuildOS, STATUS.json single status point |
| 14 | 2026-09-12 | ~350 branches accumulated (241+31 merged in monorepo alone, dupes -fresh/-v2/-rebased) | No delete-on-merge rule, no naming discipline | Branch hygiene | Cleanup executed (log: docs/pipeline-stats/branch-cleanup-2026-09.md); hygiene rules in PROCESS_FRAMEWORK.md |
| 15 | 2026-09-12 | Split-brain status answers (different agents reported different pipeline state) | Two half-status files (watchdog-state.json + pipeline-queue.json) + per-repo copies | State fragmentation | STATUS.json schema-2 single standard status point; all consumers repointed |

## Gap classes (for pattern detection)
1. **Role ambiguity** — unclear who may do what → charter + role files
2. **Tool bypass** — skipping the canonical script/tool → mandatory checklists + verification steps
3. **Concurrency** — two actors, one resource → staggered schedules, one-writer rules, lock discipline
4. **State hallucination** — reconstructing state from memory instead of reading files → cite-file rule, single status point
5. **External-system discipline** — guessing APIs / missing health gates → research-first, read-before-mutate, health checks
6. **Layer separation** — product/process/runtime mixed → three-layer architecture
7. **Branch hygiene** — unbounded branch growth → delete-on-merge, naming rules, drift audit
