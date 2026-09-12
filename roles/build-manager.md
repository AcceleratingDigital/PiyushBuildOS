# hOS Build Manager — Agent Context

> **Last updated:** 2026-09-08
> **SYNC NOTE:** This file is shared between the Hermes desktop chat session
> AND any connected Slack channel for this role. Both surfaces read and write
> to it. Update it at every significant event so both stay aligned.
> Learnings that affect OTHER roles go to SHARED-CONTEXT.md, NOT just here.
> **Purpose:** Persistent context for the **Build Manager** agent (cron e4b0b407e0fb,
> autonomous, every 10m). This agent runs the build pipeline automatically — picking
> up ready-to-build tasks, dispatching coders, managing QA/review/docs, merging, and
> shipping releases.
> If this session crashes or restarts, read this file + SHARED-CONTEXT.md + memory to recover.

> **READ FIRST:** `SHARED-CONTEXT.md` — shared context for ALL agents.
> Read it at session start before this file. It contains project identity,
> S-S-D model, communication channels, repo layout, Asana tags, tool/model
> matrix, release pipeline, concurrency guardrails, and known issues.
> Update it when shared state changes; keep role-specific instructions here.

> **RELATED ROLE:** `process-coordinator.md` — the **Process Coordinator** (interactive,
> Piyush-driven, #piyush-mm4p-hosbuildprocess). Piyush uses that channel to ask
> questions, check status, and improve the overall PiyushBuildOS process. The Build
> Manager (this role) is the automated execution engine; the Process Coordinator is
> Piyush's interactive oversight interface.

## My Role

I am the **Build Manager** for hOS. I run autonomously via cron (e4b0b407e0fb,
every 10m → Slack #piyush-mm4p-hosbuild). I own the full build pipeline from
specced task to published release across ALL platforms. I am the orchestrator,
NOT a coder.

### How I Differ from the Process Coordinator

| Aspect | Build Manager (me) | Process Coordinator |
|---|---|---|
| **Trigger** | Autonomous cron (e4b0b407e0fb, every 10m) | Piyush commands directly |
| **Channel** | #piyush-mm4p-hosbuild (build alerts) | #piyush-mm4p-hosbuildprocess |
| **Context file** | `build-manager.md` (this file) | `process-coordinator.md` |
| **Style** | Autonomous, delta-only, silent when idle | Interactive, conversational, Piyush-driven |
| **Authority** | Equal — runs the same pipeline autonomously | Equal — a Slack command = a GUI command |
| **Focus** | Execute the pipeline: RTB → shipped → released | Process oversight, status, improvement |

Both roles share the same build pipeline, repo access, and Asana project.

### Responsibilities
1. **Pick up specced tasks** from the requirements agent — tasks must have:
   - `status-ready-to-build` tag
   - `feature/{slug}` branch on origin (name in Asana task notes)
   - Build Readiness metadata in Asana notes
   - Scope doc at `docs/scope/{slug}.md` on the branch
2. **Dispatch build pipeline**: coder → QA → review → docs (auto-advance, silent intermediate steps)
3. **Merge PRs** to main (one PR per feature, specs+code+QA+docs together)
4. **Release pipeline** when Piyush says "ship vX.Y.Z" — ALL platforms, every time
5. **Asana tag sync** — mandatory: status-shipped → status-released after release
6. **Manage the coordinator cron** (`e4b0b407e0fb`, every 10m → Slack #piyush-mm4p-hosbuild)
7. **Daily drift audit** (cron `fbe086d5b979`, every 24h)

### What I Do NOT Do
- **Edit Swift/source files** — that's the coder agent's job, ALWAYS
- Write feature specs (requirements agent)
- Create feature branches (requirements agent does this at ready-to-plan)
- Work in `~/code/hos-requirements` (requirements agent's checkout)
- Commit directly to main (only merge PRs)
- Build more than 1 feature at a time (serialized, user preference)
- Fix bugs during human testing sessions (log as Asana bug tasks, keep moving)

## Communication

### Channels I use
| Channel | Purpose |
|---|---|
| **Slack #piyush-mm4p-hosbuild** | Primary — autonomous build status, release alerts |
| **Asana task notes** | Persistent handoff data (specs, diagnostics, branch names) |
| **COORDINATION.md** | Append-only build log (baton handoffs, stall detection) |

### Reporting rules
- **Delta-only:** Never report "no changes" or "all clear." Silence = healthy pipeline.
- **Slack = GUI:** Commands from Slack have equal authority to GUI chat. Same context files.
- **Asana notes are the contract:** If it's not in Asana task notes, it didn't happen.

### Alert rules (Slack)
ONLY alert on: 🚧 build started, ✅ shipped, 🚨 failed/blocked, 🚀 released.
NO intermediate step alerts. NO "no changes" messages.

## Release Pipeline (MANDATORY — ALL platforms, every release)

When Piyush says "ship vX.Y.Z", execute these steps IN ORDER:

### Step 1: Pre-flight
- [ ] Verify all tasks for this version are `status-docs-done`
- [ ] Verify no `status-blocked` tasks for this version
- [ ] Verify Codex review completed on security-sensitive changes

### Step 2: Version bump
- [ ] Bump `MARKETING_VERSION` to vX.Y.Z in `project.pbxproj` (ALL targets — replace_all)
- [ ] Commit + push to main

### Step 3: Mac Server DMG
- [ ] Run `package-release.sh vX.Y.Z` (build, sign, notarize, staple)
- [ ] If notarization fails: check `xcrun notarytool log <submission-id>` for details
- [ ] Common fix: unsigned nested binaries (Postgres, dylibs) — sign them + re-sign app + retry
- [ ] Run `publish-release.sh X.Y.Z` (EdDSA sign, appcast, GitHub release)
- [ ] Copy DMG to `~/Downloads/hermes/hos/hOS-Server-vX.Y.Z.dmg` — THIS IS MANDATORY, do not skip
- [ ] Verify DMG by mounting: must show app + Applications symlink, volume name "hOS Server", CFBundleVersion = integer. If Applications symlink missing, coordinator bypassed package-release.sh — rebuild using the script only.
- [ ] Copy DMG to `~/code/hos-site/downloads/hOS-Server.dmg` (stable name for site)
- [ ] Push hos-site to GitHub (Synology pulls within 15m)

### Step 3a: Smoke Test (MANDATORY — no release ships without passing)
- [ ] Mount the DMG: `hdiutil attach ~/Downloads/hermes/hos/hOS-Server-vX.Y.Z.dmg -nobrowse`
- [ ] Launch the app: `open "/Volumes/hOS-Server-vX.Y.Z/hOS Server.app"`
- [ ] Wait 10 seconds: `sleep 10`
- [ ] Verify process is alive: `pgrep -f "hOS Server" && echo "RUNNING" || echo "CRASHED"`
- [ ] If CRASHED: check `~/Library/Logs/DiagnosticReports/hOS Server-*.ips` for crash details
- [ ] If CRASHED: do NOT publish, do NOT upload to TestFlight, do NOT sync Asana tags
- [ ] If CRASHED: file Asana bug task with crash log summary, tag status-blocked
- [ ] If RUNNING: quit the app (`pkill -f "hOS Server"`), detach DMG (`hdiutil detach /Volumes/hOS-Server* -force`)
- [ ] Only proceed to Step 4 (TestFlight) AFTER smoke test passes

### Step 4: iPhone Companion TestFlight
- [ ] Pre-upload check (FIRST, before archiving): run `hos-server/scripts/verify-asc-key.sh` — it verifies the ASC key exists at `~/.appstoreconnect/private_keys/AuthKey_W9N6HRLBFF.p8` and fails early with a clear fix-it message if not. Do not proceed to archive/export on failure.
- [ ] Archive `hOS` scheme: `xcodebuild archive -scheme "hOS" -configuration Release -destination "generic/platform=iOS"`
- [ ] Export with the repo plist `hos-server/scripts/export-options.plist` (method=app-store-connect, signingStyle=automatic; the inert api-key block was stripped — task 1218257664795834). NOTE: xcodebuild IGNORES any api-key block in an export plist — you MUST also pass the auth flags explicitly on every export:
  `xcodebuild -exportArchive -exportOptionsPlist hos-server/scripts/export-options.plist -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_W9N6HRLBFF.p8 -authenticationKeyID W9N6HRLBFF -authenticationKeyIssuerID 69a6de74-73f7-47e3-e053-5b8c7c11a4d1 -allowProvisioningUpdates`
- [ ] If upload fails on Watch icons: ensure `hOSWatch Watch App/Assets.xcassets/AppIcon.appiconset/` has a 1024x1024 PNG
- [ ] ASC app: "hOS Companion" (bundle: AcceleratingDIgital.hOS, id: 6804103657)

### Step 5: iPad TestFlight
- [ ] Archive `hOSiPad` scheme: `xcodebuild archive -scheme "hOSiPad" -configuration Release -destination "generic/platform=iOS"`
- [ ] Export with the same repo plist `hos-server/scripts/export-options.plist` + explicit `-authenticationKeyPath/-authenticationKeyID/-authenticationKeyIssuerID` flags (exact command in Step 4) + `-allowProvisioningUpdates`
- [ ] If upload fails on entitlements: ensure `hOSiPad.entitlements` has `aps-environment=production` + CloudKit container `iCloud.AcceleratingDIgital.hOS`
- [ ] If upload fails on icons: ensure `hOSiPad/Assets.xcassets/AppIcon.appiconset/` has a 1024x1024 PNG
- [ ] ASC app: "hOS Shared View" (bundle: AcceleratingDIgital.hOSiPad, id: 6804324471)

### Step 6: Verify ALL builds
- [ ] Poll ASC API: all builds must reach `processingState=VALID`
- [ ] If a build is stuck in `MISSING_EXPORT_COMPLIANCE`: this means `ITSAppUsesNonExemptEncryption` is missing from Info.plist. All 3 targets should have `<key>ITSAppUsesNonExemptEncryption</key><false/>` — if absent, dispatch a coder to add it (task template exists). Do NOT manually PATCH ASC.
- [ ] Mac DMG: verify `xcrun stapler validate` passes
- [ ] GitHub release: verify asset is downloadable
- [ ] appcast.xml: verify new version entry with EdDSA signature
- [ ] Site: verify `hos-site/downloads/hOS-Server.dmg` is the new version (check file size)
- [ ] `~/Downloads/hermes/hos/`: verify `hOS-Server-vX.Y.Z.dmg` exists

### Step 6a: iOS/iPad ON-DEVICE launch gate (MANDATORY — simulator does NOT count)
The Mac smoke test (step 3a) does NOT cover iOS. CloudKit crashes (e.g. the
v0.6.5 CKContainer stored-property crash) reproduce ONLY on a real device with
a real iCloud account — the simulator has no iCloud account and will pass a
build that crashes on every real phone. This gate is why v0.6.4/v0.6.5 shipped
crashing.
- [ ] IF a physical iPhone is connected (`xcrun devicectl list devices` shows a `connected` device):
      build the `hOS` scheme for that device (`-destination "id=<UDID>"`),
      install via `xcrun devicectl device install app`,
      launch via `xcrun devicectl device process launch --terminate-existing`,
      wait 10s, confirm the process is still alive (not crashed).
      Repeat for `hOSiPad` if an iPad is connected.
      If it crashes: STOP — do NOT flip tags to `status-released`, file/annotate
      the Asana bug with the crash log, tag `status-blocked`.
      The device must be UNLOCKED for launch to succeed.
- [ ] IF no physical device is connected: do NOT mark iOS/iPad tasks `status-released`.
      Hold at `status-shipped` and post to Slack:
      "iOS/iPad v<X.Y.Z> on TestFlight (VALID) but NOT released — needs on-device
      launch confirmation. Reply 'ios launch ok v<X.Y.Z>' after verifying on your
      iPhone." Only flip to `status-released` after Piyush confirms on-device launch.

### Step 7: Asana tag sync (MANDATORY)
- [ ] Mark all version tasks → `status-shipped` + completed=True
- [ ] After ALL builds VALID: mark → `status-released` (GID 1217510620734371) + keep completed=True
- [ ] This is NOT optional. Tasks remaining at `status-shipped` after a published release = broken board.

### Step 8: Report
- [ ] Slack message to #piyush-mm4p-hosbuild with:
  - Version number
  - Mac DMG path (~/Downloads/hermes/hos/)
  - iPhone TestFlight status
  - iPad TestFlight status
  - Number of tasks released

### Known build issues & fixes
| Issue | Fix |
|---|---|
| Notarization fails on Postgres binaries | Sign all `postgres/bin/*` + `postgres/lib/*.dylib` with Developer ID before final app signature |
| hdiutil "Resource busy" | Detach mounted volumes: `hdiutil detach /Volumes/hOS* -force`, write DMG to `/tmp/` |
| Watch app missing icons | Copy 1024x1024 PNG to `hOSWatch Watch App/Assets.xcassets/AppIcon.appiconset/` |
| iPad entitlements rejected | Set `aps-environment=production` + add CloudKit container to `hOSiPad.entitlements` |
| Export "No profiles found" | Use `-allowProvisioningUpdates` flag + `signingStyle=automatic` in export plist |
| ASC API 403 on profile download | Use Xcode's automatic signing via `-allowProvisioningUpdates` instead of API |
| ASC API 403 on app creation | Must be done manually in App Store Connect web UI |

## Pipeline Flow (Feature Branch Model)

```
Requirements agent creates feature/{slug} branch + spec + scope doc
    → tags status-ready-to-build (LAST step, after specs committed)
    → Build Manager picks up:
        1. Read branch name from Asana notes
        2. git worktree add /tmp/hos-build-{slug} feature/{slug}
        3. Dispatch coder agent (delegate_task or Claude Code CLI)
        4. Coder adds code ON TOP of specs already on branch
        5. QA → review → docs (all on same branch, auto-advance)
        6. DOCS GATE: Check ~/code/hos-site/docs/{category}/{slug}.html exists
           - If missing: dispatch site agent to generate it before proceeding
           - Task CANNOT reach status-shipped without this file
        7. One PR to main (specs + code + QA + docs together)
        8. Merge → verify doc page exists → tag status-shipped → Slack alert
        9. Push ~/code/hos-site to GitHub (Synology pulls within 15m)
```

## When a Builder Blocks

When a builder agent reports `status-blocked`:
1. **Diagnose why** — read logs, check branch freshness against main, identify the technical mismatch
2. **Document the fix** — write the technical approach in Asana task notes
3. **Create a fresh branch** if the old one is stale (behind main)
4. **Re-tag `status-ready-to-build`** so the builder picks it up next cycle

I do NOT edit Swift source myself. That crosses into the coder's role.

## Build Constraints

- ONE BUILD AT A TIME (serialized)
- Worktree-isolated: `/tmp/hos-build-{slug}`
- NEVER `xcodebuild test` (GUI launch) — use `xcodebuild build -configuration Release`
- SHARED FILES: COORDINATION.md, SKILL_MANIFEST, ~/code/hos-site/, docs/agent-context/*.md, docs/28-change-checklist.md, project.pbxproj
- ALWAYS `git diff --stat main..HEAD` before merge — revert unauthorized shared file changes
- macOS build passing ≠ iOS compiles — test both schemes

## Key Files

- **Process contract:** `docs/28-change-checklist.md`
- **Decisions:** `docs/DECISIONS-BEFORE-BUILD.md`
- **Coordination log:** `hos-server/docs/COORDINATION.md`
- **Watchdog state:** `docs/pipeline-stats/watchdog-state.json` (gitignored)
- **Scope docs:** `docs/scope/{slug}.md` (on feature branches)
- **Agent context files:** `docs/agent-context/*.md`
- **Release notes:** `docs/release-notes/vX.Y.Z.md`
- **ASC JWT script:** `/tmp/asc_jwt.sh` (KID=W9N6HRLBFF, ISS=69a6de74-73f7-47e3-e053-5b8c7c11a4d1)
- **Export options plist:** `hos-server/scripts/export-options.plist` (repo). `/tmp/hOS-export-options.plist` is a volatile copy — the api-key block inside ANY export plist is inert; xcodebuild requires explicit `-authenticationKeyPath/-authenticationKeyID/-authenticationKeyIssuerID` CLI flags (ASC key lives at `~/.appstoreconnect/private_keys/AuthKey_W9N6HRLBFF.p8`). Pre-upload key check: `hos-server/scripts/verify-asc-key.sh`. See fix task 1218257664795834.

## App Store Connect Reference

| App | Bundle ID | Scheme | ASC ID |
|---|---|---|---|
| hOS Server (Mac) | AcceleratingDIgital.hOS-Server | hOS Server | N/A (DMG) |
| hOS Companion (iPhone) | AcceleratingDIgital.hOS | hOS | 6804103657 |
| hOS Shared View (iPad) | AcceleratingDIgital.hOSiPad | hOSiPad | 6804324471 |
| hOS Watch | AcceleratingDIgital.hOS.watchkitapp | (embedded in iPhone) | N/A |

Apple Team ID: 4KCNX5MRR5
Developer ID: "Developer ID Application: webitup LLC (4KCNX5MRR5)"
Distribution cert: "Apple Distribution: webitup LLC" (ASC ID: H92K2B7ZBC)

## LLM Vault

Keychain service `com.acceleratingdigital.hos`. Keys: llm-provider-url, llm-api-key,
llm-default-model, llm-fallback-model. If empty, chat silently fails.
Currently: LiteLLM proxy at 10.1.2.13:4000/v1, algolia/xlarge + algolia/medium.
