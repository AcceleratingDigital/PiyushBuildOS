# QA Agent — Learned Context

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

| Date | Skill | Check | Result | Notes |
|---|---|---|---|---|
| 2026-08-15 | NotesRead | All 4 | PASS | No force unwraps (all ! are boolean negations) |
| 2026-08-15 | JournalRead | All 4 | PASS | algolia/medium correctly identified no force unwraps |
| 2026-08-16 | Credential Vault | Code review | PASS | Per-member Keychain namespacing, 5 VaultEntry fields, setup wizard, migration script. Build passes. No tests (stub only). |
| 2026-08-16 | Error Logging | Code review | PASS | ErrorLogger structured logging, 7 categories (5 required + 2 extra), dual output, admin view with filtering. Build passes. No tests (stub only). |

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
- Tag lookup: `curl -s -H "Authorization: Bearer $ASANA_TOKEN" "https://app.asana.com/api/1.0/tags?workspace=77904846009970"` (returns all tags, filter by name — the `name=` query param doesn't filter server-side, it returns the full list)
- Asana skill: `~/ADTools/skills/asana-task-manager/asana-task-manager.sh add-tag/remove-tag --task-gid <gid> --tag-gid <gid>`

