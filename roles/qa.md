# QA Agent — Learned Context

> **SYNC NOTE:** This file is shared between all surfaces that run QA agents.
> Update it at every significant event so future runs stay aligned.
> Learnings that affect OTHER roles (coder, review, build) go to SHARED-CONTEXT.md, NOT just here.

Read this file before starting any QA verification. It accumulates test
patterns, edge cases found, and build verification steps.

## QA checklist (always verify)

1. **Source review:** Read the Swift source. Check:
   - `@objc(<Name>Skill)`, `@MainActor`, `NSObject`, `Skill` protocol present
   - Manifest has id, name, version, capabilities, inputs
   - Correct Domain and Action used
   - Inputs match spec (required vs optional, defaults, validation)
   - Graceful degradation (returns message, doesn't crash)
   - Force unwraps (`!`) — list each and assess crash risk
2. **Build verify:** Run `build-skill.sh` — must produce `.bundle`
3. **Bundle verify:** Check bundle has `MacOS/<Name>`, `Info.plist` with `NSPrincipalClass`, `_CodeSignature`
4. **Code quality:** Error handling, limit validation, input sanitization

## Build commands

```bash
cd ~/code/hos-monorepo/hos-server
CONFIGURATION=Debug ./scripts/build-skill.sh <Name> <Name> <HOS<Name>Skill> com.acceleratingdigital.hos.skill.<name> <version>
```

## Edge cases to test

- Empty result set (query matches nothing)
- Missing dependency (vault/app not found)
- Permission denied (no TCC access)
- Large input (1000+ entries)
- Special characters in query (*, [], unicode)
- Limit edge cases (0, negative, > 100)
- Null/empty inputs

## File locations (IMPORTANT — search these paths, not repo root)

- `SKILL_MANIFEST` is in `hos-server/scripts/package-release.sh` (NOT repo root)
- `COORDINATION.md` is at `hos-server/docs/COORDINATION.md` (NOT repo root)
- `STATE.md` is at repo root `~/code/hos-monorepo/STATE.md`
- `hOS-Server-Info.plist` is at `hos-server/hOS-Server-Info.plist` or `hos-server/hOS Server/hOS-Server-Info.plist`
- Skill source: `hos-server/skills/<Name>/<Name>Skill.swift`
- Skill bundles: `hos-server/skills/<Name>/build/Debug/<Name>.bundle`
- Agent context files: `docs/agent-context/`
- Process contract: `docs/28-change-checklist.md`
- Pipeline stats: `docs/pipeline-stats/`

**When a grep returns no matches, try the other path before reporting FAIL.**
False positives waste pipeline time.

|| Date | Skill | Check | Result | Notes |
||---|---|---|---|---|
|| 2026-08-15 | NotesRead | All 4 | PASS | No force unwraps (all ! are boolean negations) |
|| 2026-08-15 | JournalRead | All 4 | PASS | algolia/medium correctly identified no force unwraps |
|| 2026-08-16 | Credential Vault | Code review | PASS | Per-member Keychain namespacing, 5 VaultEntry fields, setup wizard, migration script. Build passes. No tests (stub only). |
|| 2026-08-16 | Error Logging | Code review | PASS | ErrorLogger structured logging, 7 categories (5 required + 2 extra), dual output, admin view with filtering. Build passes. No tests (stub only). |
|| 2026-08-18 | stale-doc-cleanup | Docs QA | PASS | Verified 7 acceptance criteria: E2/D3/F4 stale refs fixed, emitKnowledge 3-param, gateway clarified, model names marked, 5+ files updated, no regressions, 3+ commits. Git diff shows all changes. |

## Force unwrap detection

`!` in Swift can be:
- **Boolean negation:** `!query.isEmpty` — SAFE
- **Force unwrap:** `optional!` — RISKY (crashes on nil)
- **Force cast:** `as!` — RISKY

Distinguish by context. `!` after a variable name is force unwrap. `!` before
a boolean expression is negation.

## QA patterns for infrastructure features (non-skill)

Infrastructure features (CredentialVault, ErrorLogger, etc.) follow a different
QA pattern than skills:

1. **No bundle to verify** — these are compiled into the server app target, not
   packaged as `.bundle` skills. Skip build-skill.sh and bundle checks.
2. **Build verification:** `xcodebuild build -project "hOS Server.xcodeproj" -scheme "hOS Server" -configuration Debug` — must show `** BUILD SUCCEEDED **`
3. **Test targets are stubs:** `hOS_ServerTests.swift` and `SkillKitTests.swift`
   contain only `example()` placeholder tests. No real test coverage exists for
   any feature as of 0.5.0. This is a **gap to note, not a FAIL**.
4. **xcodebuild test is slow** (>10 min for this project). If tests are stubs,
   don't wait indefinitely — kill after ~3 min and note the gap.
5. **Admin views may not be in ChatAdminViews.swift:** The spec said "ChatAdminViews.swift
   (admin view)" but the Error Log admin view (`HealthView`/`ErrorLogPanel`) was
   actually in `hOS Server/ContentView.swift`. Always search the whole `hos-server/`
   directory for view names, not just the file the spec mentions.
6. **Commit scope vs spec:** A commit may touch fewer files than the spec lists.
   The Error Logging commit (df6b132) only added ErrorLogger.swift + ErrorLog.swift;
   the admin view was in the Credential Vault commit (96272a2). Check both commits.
7. **Sensitive values redacted in file reads:** `apiKey` fields show as `***` in
   read_file output — this is expected, not corruption.

## Asana tag workflow

- `status-ready-for-qa` GID: 1217508257505680
- `status-qa-passed` GID: 1217508019549179
- `status-blocked` GID: 1217507685607284
- Tag lookup: `curl -s -H "Authorization: Bearer ***" "https://app.asana.com/api/1.0/tags?workspace=77904846009970"` (returns all tags, filter by name — the `name=` query param doesn't filter server-side, it returns the full list)
- Asana skill: `~/ADTools/skills/asana-task-manager/asana-task-manager.sh add-tag/remove-tag --task-gid <gid> --tag-gid <gid>`

## Lessons learned — Docs-only QA
> **READ FIRST:** `SHARED-CONTEXT.md` — shared context for ALL agents.
> Read it at session start before this file. It contains project identity,
> S-S-D model, communication channels, repo layout, Asana tags, tool/model
> matrix, release pipeline, concurrency guardrails, and known issues.
> Update it when shared state changes; keep role-specific instructions here.



**For docs-only tasks (no code build):**
- `git log --oneline main..HEAD` shows commits to verify; >= 3 commits is typical for multi-file doc fixes
- `git show <commit-sha>` diff shows **exactly** what was fixed — use this as your acceptance criteria checklist
- Verify 5+ file modifications by checking git stat; read each modified file to spot-check the fixes
- API naming changes (categorize→classify) appear in multiple locations; search broadly (all scope docs, not just one)
- Decision supersedures (F4) are marked explicitly with **SUPERSEDED** tags; check for these in decision docs
- Stale references live in decision documents alongside current decisions; both old and new may coexist — make sure the OLD reference is updated to the NEW one
- Gateway docs often clarify implementation choices (Network.framework vs Vapor); look for DECISION CLARIFICATION sections
- Model manifests include example notes; these are often added as NOTEs mid-section, not always at the top

## Lessons learned — D5 Brief Consolidation QA (2026-08-20)

- D5 build verify: `xcodebuild -project "hOS Server.xcodeproj" -scheme "hOS Server" -derivedDataPath /tmp/<worktree>/DerivedData -configuration Release build` — PASSES (Release).
- When a refactor splits a monolithic method (LLMService.generateBrief) into an orchestrator (BriefGenerator) + sub-components, trace the FULL data flow end-to-end: the legacy path stored actionItemIDs in loop metadata BEFORE creating the loop; the new path parks approvals AFTER creating the loop and DISCARDS the IDs — the companion API reads metadata["actionItems"] and gets nothing.
- BriefGenerator.generate() order-of-operations bug: storeBriefLoop (line 102) runs before parkActionItems (line 108), but parkActionItems never writes the approval IDs back into the loop metadata. Comment at BriefGenerator.swift:131-132 ("will be updated by parkActionItems") is aspirational — the merge never happens. Compare with legacy LLMService.swift:639-641 which sets metadata["actionItems"] before createLoop.
- Provider priority verification: EmailBriefProvider.priority=70, Finance=60, Calendar=50, Chore=40 — sorted descending in BriefProviderRegistry.gatherSections (line 146). Weather provider is O1 (not yet shipped) so the "weather doesn't override urgent email" DoD is structurally satisfied by the priority sort, not runtime-tested.
- Graceful degradation pattern is consistent across all 4 providers: each `contribute` method has `guard let checkpoint else { return nil }` + do/catch returning nil; BriefProviderRegistry.gatherSections compactMaps nils; BriefSynthesizer has empty-sections fallback (line 52) + LLM-failure fallback (line 128).
- Deprecated skill verification: search SkillScheduler.swift for skill IDs "hello-household" / "morning-status" → 0 matches. Scheduler seeds "com.acceleratingdigital.hos.skill.daily-brief" (SkillScheduler.swift:453-466). Deprecated skills retain DEPRECATED header comments but are not registered/scheduled.
- DailyBriefSkill.swift was NOT modified to call BriefGenerator directly — it still calls `context.generateBrief(member:)` (line 65), which routes through SharedCapabilities → LLMService.generateBrief → briefGenerator delegation (LLMService.swift:512-515). This is transparent delegation, acceptable but worth noting: the skill file's header comment still describes the old tool-loop, not the D5 delegation.

## Lessons learned — O4 Family Location Sharing QA (2026-08-20)

- O4 build verify: `xcodebuild -project "hOS Server.xcodeproj" -scheme "hOS Server" -derivedDataPath /tmp/<worktree>/DerivedData -configuration Release build` → ** BUILD SUCCEEDED ** (macOS). iOS scheme "hOS" → ** BUILD SUCCEEDED **. Both pass.
- Privacy enforcement (E3) lives at the PolicyCheckpoint broker seam, NOT the store. `familyLocations()` and `locationBrief()` both compute `let includePrivate = who.isOwner` and pass it to `locationStore.snapshotsAsDictionaries(includePrivate:)`. When false, members with `sharingEnabled=false` are filtered out. This is the correct architecture — the store is neutral, the policy layer enforces.
- GeofenceMonitor uses `CLCircularRegion` (not CLRegion directly) with `locationManager.startMonitoring(for:)` — correct CoreLocation API. Push is via CloudKit `CKQuerySubscription` (predicate `type == "geofence"`) in PushNotifications.swift, NOT custom APNs. CloudMailbox.writeGeofenceAlertOutbox writes the CKRecord. This matches the spec.
- LocationPatternAnalyzer: groups snapshots by (label, day-of-week), computes median arrival (first snapshot/day) and departure (last snapshot/day) times. 3-sample noise filter. 30-day window. Uses `Calendar.current` — server-side, correct. The analyzer is a `@MainActor struct` invoked by SkillScheduler's `runLocationPatternAnalysis()` daily task (1440 min interval).
- LocationBriefProvider: `@MainActor final class` conforming to `BriefProvider`. `contribute(member:date:)` returns `BriefSection?`. `contribute()` returns String. Generates "Family locations: Maya is at school (since 2 hr ago), ETA home: 3:30 PM; Home is empty." ETA estimated from LocationPattern typicalDeparture for current weekday + label. Priority 50 (same as Calendar). Graceful degradation: returns "" if store nil or no snapshots.
- FamilyMapView (iOS): MapKit `Map(position:)` with `Annotation` for each member. Parses location from brief detail text (not a direct API) — this is a V1 limitation noted in code. Uses `@Environment(HOSClient.self)`. Hardcoded member list in LocationSettingsView (`["Owner", "Maya", "Dad", "Mom", "Alex"]`) — V1 placeholder, not a FAIL but worth noting for production.
- SKILL_MANIFEST entry: `"FamilyLocation|FamilyLocation|HOSFamilyLocationSkill|com.acceleratingdigital.hos.skill.family-location|0.1.0"` at line 94 of package-release.sh. Confirmed.
- No `.location` Domain added to SkillKit — FamilyLocationSkill uses `.household` domain (`.read` + `.mutate`). This is acceptable; location is a household-domain capability.
- JSON injection risk (LOW): LLMService.swift:2583 constructs JSON via string interpolation: `"{\"summary\": \(escaped), \"data\": \(json)}"`. The `escaped` variable is meant to be JSON-encoded but `JSONSerialization.data(withJSONObject: summary, options: [.fragmentsAllowed])` fails for String input (expects Array/Dict), so `escaped = summary` (raw). If summary contains a `"`, JSON is malformed. Since summary is internally generated and unlikely to contain quotes, this is LOW severity. Same pattern exists in the tutor_brief tool — not a regression.
- No force unwraps in any O4 new files (LocationStore, GeofenceMonitor, LocationPatternAnalyzer, LocationBriefProvider, FamilyMapView, LocationSettingsView, FamilyLocationSkill). The one force unwrap found (`version!` in HOSDoctor.swift:191) is pre-existing, not from this branch.
- No `requestPermission` (wrong API) found — GeofenceMonitor uses `locationManager.requestAlwaysAuthorization()` (correct). PushNotifications uses `UNUserNotificationCenter.current().requestAuthorization(options:)` (correct). No iOS permission API misuse.
- No unescaped quotes in SwiftUI `Text()` in new iOS files. LocationSettingsView line 149 has escaped quotes in a Text string literal — correctly escaped.
- DerivedData is in .gitignore and not committed. `git diff --stat main..HEAD -- DerivedData/` returns empty.
- Shared file changes (11 files) are all O4-related: LLMService (location_brief tool + late-inject), PolicyCheckpoint (9 broker methods), PostgresSchema (4 tables), SkillKit (9 broker + 9 convenience methods), SkillScheduler (pattern analysis task), CloudMailbox (writeGeofenceAlertOutbox), PushNotifications (geofence CKQuerySubscription), FamilyDashboardView both targets (.location card), hOS_ServerApp (wiring), package-release.sh (manifest entry). COORDINATION.md append-only (2 lines). No unauthorized changes.

## Lessons learned — PostgreSQL Not Starting QA (2026-08-22)

- **dylibbundler -od wipes the output directory** — the `-od` (overwrite-dir) flag totally removes the target dir before writing. If you copy files into the dir BEFORE running dylibbundler (as bundle-postgres.sh does with vector.dylib at line 98 before dylibbundler at line 134), those files are destroyed. FIX: copy vector.dylib AFTER all dylibbundler runs, or use `-of` (overwrite-files) instead of `-od`.
- **pgvector vector.dylib missing from bundle** — confirmed vector.dylib builds correctly at /tmp/pgvector-build/vector.dylib (201KB) but never reaches the app bundle's Resources/postgres/lib/ due to the dylibbundler -od ordering bug above. CREATE EXTENSION vector will fail at runtime. The .control and SQL migration files ARE present in share/extension/ (they go to a different directory not touched by dylibbundler).
- **Bundle verification must check ALL expected artifacts** — not just that the directory exists, but each specific binary/file (postgres, initdb, pg_dump, vector.dylib, vector.control, SQL files). Missing vector.dylib was caught only by exhaustive `find` + `ls` of the lib directory.
- **git diff --stat main..HEAD can show misleading "changes"** when the branch base is behind main. The build-coordinator.md "regression" was actually just the branch having an older version than main (main advanced past the branch base). Always check `git log --oneline main..HEAD -- <file>` to see WHICH commits on the branch touched shared files, and separate requirements-agent commits from builder commits.
- **waitForRunning pattern verification**: search for `Task.sleep.*15` or `Task.sleep.*10[^0]` (not followed by more zeros, to distinguish 10s from 100s/1000s) to confirm fixed sleeps were removed. Zero matches = all replaced with waitForRunning.
- **ErrorLogger integration verification**: grep for `ErrorLogger.shared.log` + `severity: .critical` in the same function to confirm startup failures are logged at critical severity. PostgresManager has 2 call sites (findBinDirectory nil + startup catch block), both with `.critical`.

## Lessons learned — Calendar Event Creation Fails QA (2026-08-22)

- **EventKit auth status dual-check pattern**: `guard status == .fullAccess || status == .authorized` — must accept BOTH on macOS 14+ (.fullAccess) and pre-14/legacy (.authorized). The reminders provider already does this; calendar provider was missing .authorized. Always grep BOTH createEvent AND fetchEvents guards.
- **Nil defaultCalendarForNewEvents fallback**: `store.defaultCalendarForNewEvents ?? store.calendars(for: .event).first(where: { !$0.allowsContentModifications == false })` — double-negative on allowsContentModifications is intentional (ObjC BOOL bridging). Fallback prevents nil-calendar save crash.
- **EventKit error wrapping**: bare `catch { throw error }` loses the real description to the LLM as generic "technical error". Must wrap: `throw SkillError.writeNotSupported("...: \(error.localizedDescription)")`.
- **Audit-after-write ordering**: when auditAllow() runs AFTER a successful write, a Postgres-down failure must NOT surface as generic "access denied". Capture the created id, catch audit failure separately, throw `writeNotSupported("event created but audit failed")`.
- **isAuditFailure() helper** (PolicyCheckpoint.swift:1320) matches `.auditWriteFailed` only — verify this is the case before surfacing the "event created" message.
- **Missing-argument errors should instruct the LLM**: `throw SkillError.missingArgument("title — ask the person what to call the event")`. Start-time error must say "do not default to today".
- **Duration fallback semantics**: manifest description should say "falls back to 60 only when explicitly omitted" and result summary should include the duration used.
- **os.Logger verification**: grep for `log.error(` in DataProviders — must log auth status raw value, nil calendar case, and save error.
- **Build verification for bug-fix branches**: use `-configuration Release` (NOT Debug — Debug launches GUI). Both macOS (scheme "hOS Server") and iOS (scheme "hOS", destination "generic/platform=iOS") builds must pass.

## Lessons learned — Approvals Not Syncing to iOS QA (2026-08-22)

- **Diagnostics-only Phase 1 + resilience Phase 3 pattern**: this bug fix is purely additive (no behavioral changes to sync logic). QA must verify the diagnostic surfaces exist and are wired correctly, NOT that sync itself works (that's Phase 2, pending runtime findings).
- **@Observable migration**: CloudMailbox was converted to `@Observable` with a `CloudMailboxDiagnostics` struct. Verify `@Bindable` usage in SwiftUI views (CloudSyncStatusCard uses `@Bindable var mailbox: CloudMailbox` — correct for `@Observable` objects in SwiftUI).
- **CKError structured logging verification**: search for `CKError code=` in os_log calls — both fetchRecord and writeRecord in CloudMailbox.swift now log `ckErr.code.rawValue` + `String(describing: ckErr.code)`. CloudApprovals.swift mirrors this pattern.
- **CloudKit environment detection**: `readCloudKitEnvironment()` reads the APNs entitlement from the code signature (`aps-environment`). On iOS, `SecStaticCode`/`SecCodeCopySigningInformation` are unavailable, so it returns nil → "unknown". This is expected — the environment matters most on the Mac (writer side).
- **testSync() probe pattern**: write a marker to a dedicated `hos-approvals-test-sync` record (not the real outbox), read it back, delete it. This exercises the exact fetch/save/delete path without perturbing live approvals state. Verify the probe record name is distinct from the real outbox.
- **ContentView.swift change is expected**: iOS ContentView passes `cloudApprovals` to `ConnectivityDiagnostics.runAll()` — this is the call-site wiring for the new `cloudApprovals` parameter, not an unauthorized change. Always check if a file change is the necessary wiring for a parameter added in an expected file.
- **DiagnosticsPanel → HOSDoctor cloudMailbox wiring**: DiagnosticsPanel (in DiagnosticBundleExporter.swift) reads `@Environment(CloudMailbox.self)` and passes it to `doctor.runFullCheck(members:cloudMailbox:)`. Verify the overload chain: `runFullCheck()` → `runFullCheck(members:)` → `runFullCheck(members:cloudMailbox:)`.
- **checkOutboxHealth in HOSDoctor**: delegates to `CloudMailbox.checkOutboxHealth()` which returns an `OutboxHealthResult` struct with `.ok/.warn/.fail` status. HOSDoctor maps these to DiagnosticEntry statuses. Verify the mapping is correct (ok→pass, warn→warn, fail→fail).
- **CloudApprovals reload() fetch result categories**: verify all 5 categories are set: `found`, `notFound`, `parseFailed`, `noAccount`, `error`. Each must have a corresponding os_log call. The `lastFetchResult` field is read by ConnectivityDiagnostics.checkApprovalsSync to surface the specific failure mode.

---

## Lessons learned — Postgres Lifecycle Fix QA (2026-09-07)

- xcodeproj is NOT at repo root — it's at `hos-server/hOS Server.xcodeproj`. Run xcodebuild with workdir `hos-server` and `-derivedDataPath ../DerivedData`; a root-level invocation fails with "does not exist".
- `await` on a non-async func compiles in this project's Swift mode (warning only) — e.g. `await Self.runTool(...)` where runTool returns Bool. Don't flag as build-breaking; it's a blocking `waitUntilExit()` on the actor thread.
- Restart-after-stop race check: when a fix stops a process that has a `terminationHandler`, verify the crash-restart handler's state guard (e.g. `guard case .running`) is checked in the SAME actor turn that sets `.stopped` — no suspension point between the await-ed stop call and the state write means the guard can't see stale `.running`.
- Unlaunched `Process()` reports `processIdentifier == 0` (not -1); `terminate()` on it raises NSInvalidArgumentException. Sentinel-process pattern is safe only if every consumer checks `pid <= 0`.
- postmaster.pid real layout (differs from some spec docs): line 1=pid, line 2=data dir, line 3=start time, line 4=port, line 5=socket dir. Parser must use lines[1] for data dir and lines[3] for port.
- postmaster.pid data-dir comparison is plain string equality against the `-D` path the app passed — safe because postgres writes back exactly what it was launched with; but any future path normalization (resolvingSymlinks) must be applied to BOTH sides or adopt-path triage false-positives to "foreign instance".
- Gap found (minor, out of stated scenarios): a pid file whose line 1 is unparseable garbage → `PostgresPIDFile.read()` returns nil → treated as .fresh → the corrupt file is NOT removed and postgres itself fails with "invalid data in PID file". A corrupt-pid-file cleanup branch would close this.
- Graceful-stop timeout budgeting: pg_ctl `-t 10` + SIGTERM 5s wait = 15s worst case, exactly equal to the app-terminate semaphore cap — the cap can expire right as the fallback finishes. Keep the semaphore cap strictly greater than the sum of inner timeouts.

---

## Lessons learned — Postgres Lifecycle Rework RE-QA (2026-09-07)

- Re-verification pattern: validate each review finding against the `git show <commit>` diff's exact mechanism (proc_pidpath + start-time match fail-closed, SHOW data_directory vs data-dir comparison with pathsMatch normalization, setsockopt SO_RCVTIMEO) — not just "code looks plausible".
- xcodebuild Release build emits scary DTDK "device is passcode protected" remote-device stack traces that are unrelated noise; judge build success solely on the final `** BUILD SUCCEEDED **` line.
- classifyPort fail-safe defaults: socket-creation failure and send failure are classified conservatively (.foreignService / .postgresLike) so the launch path surfaces the real error rather than silently proceeding — verify reworked classification branches, not just the happy path.
- Committing between code commit and re-QA (e.g. docs-only state commits like 03a3038a) is fine for re-verification — confirm the later commits touch no source (`git show --stat`) so the worktree still equals the reviewed commit.

---

## Event-Driven Pipeline: Mandatory Final Steps

**These calls are mandatory when running under the event-driven pipeline (HOS_TASK_GID is set).**

On SUCCESS (QA PASS) — call as your absolute last action:
```bash
~/ADTools/skills/hos-pipeline-trigger/trigger.sh \
  --task-gid "$HOS_TASK_GID" \
  --branch "$HOS_BRANCH" \
  --scope-doc "$HOS_SCOPE_DOC" \
  --completed-step qa \
  --next-step reviewer-security \
  --summary "[one-line QA result, e.g. QA PASS: build clean, no force unwraps, bundle complete]"
```

On FAILURE (QA FAIL) — call before exiting:
```bash
~/ADTools/skills/hos-pipeline-trigger/fail.sh \
  --task-gid "$HOS_TASK_GID" \
  --branch "$HOS_BRANCH" \
  --failed-step qa \
  --reason "[what failed, e.g. build error in SkillKit/File.swift:42 — force unwrap on optional]" \
  --needed "[what human needs to do, e.g. fix force unwrap, push to feature/slug, then reply 'retry qa <task-gid>']"
```

**If HOS_TASK_GID is not set** (old cron-driven invocation): behave as before — no trigger or fail call needed.

Pipeline envelope variables injected by trigger.sh: HOS_TASK_GID, HOS_BRANCH, HOS_SCOPE_DOC, HOS_COMPLETED_STEP, HOS_NEXT_STEP, HOS_SUMMARY, HOS_STARTED_AT.
- 2026-09-07 QA (r2 73a12a59, task 1218226158100466): fail-closed PID validation + mutation-connection reverify + 2s start-time tolerance + 5s proof deadline all verified in source and Release build PASS; verify fixes by reading the diff at the exact commit, not HEAD state alone.
- 2026-09-07 QA r3 (a88b0746, task 1218226158100466): pg_ctl gate now covers tracked-child + malformed-pid paths, 5s deadline via lock-protected one-shot box + 100ms polling (provably returns), defer-style conn.close on both error paths; Release build PASS — 3/3 fixes verified, QA-PASS.
- 2026-09-07 QA (r4 rework, task 1218226158100466): PASS — pidFileFailedValidation defaults true (fail-closed, PostgresManager.swift:340); pg_ctl gate at :405-406 allows only validated pid file or explicit fresh-start (noPidFileExisted && classifyPort==.free); malformed/recycled pid files all fail closed (:361,:370); classifyPort conservative fallbacks (:913,:941) keep .free unreachable for unknown listeners; diff touches only PostgresManager.swift + coder.md.
