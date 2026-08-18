# Coder Agent — Learned Context

Read this file before starting any coding task. It accumulates patterns,
conventions, and pitfalls specific to hOS development.

## SkillKit conventions

- Every skill is a Swift class: `@objc(<Name>Skill)`, `@MainActor`, `NSObject`, `Skill` protocol
- Import `SkillKit` (not `HOS` or `Server`)
- Manifest: `SkillManifest(id:name:version:capabilities:inputs:)`
- Capabilities: `Capability(domain: .<domain>, action: .<action>)`
- Available domains: `.calendar`, `.contacts`, `.mail`, `.messages`, `.files`, `.system`, `.finance`
- Available actions: `.read`, `.write`, `.search`, `.send`
- **No `.notes` or `.journal` domain** — use `.files` with `.read` for these
- Return `SkillResult` (success with text) or `.failure` with message
- Skill entry point: `func perform(with context: HostContext) async throws -> SkillResult`

## Build commands

```bash
cd ~/code/hos-monorepo/hos-server
CONFIGURATION=Debug ./scripts/build-skill.sh <Name> <Name> <HOS<Name>Skill> com.acceleratingdigital.hos.skill.<name> <version>
```

## Patterns that work

- **JXA via Process:** For Apple apps without Swift frameworks (Notes, Reminders),
  use `Process()` to run `osascript -l JavaScript -e '<JXA>'`. See `NotesReadSkill.swift`.
  **WARNING: JXA is SLOW** — see Performance lessons below. Only use when no faster path exists.
- **FileManager for file-based data:** For Obsidian/markdown, use `FileManager` directly.
  See `JournalReadSkill.swift`.
- **Graceful degradation:** Always return a user-friendly message on failure, never crash.
  Use `guard let` + return `.failure("X not available")` pattern.
- **Limit clamping:** Always clamp `limit` input to 1-100. Reject 0/negative.

## Performance lessons (from real testing)

These are proven performance facts about hOS data access — follow them, don't rediscover:

1. **JXA is slow for reads.** osascript spawning + AppleScript/JS bridge overhead
   is significant. For data that has a SQLite backing store, ALWAYS prefer direct
   SQLite access over JXA. JXA is the fallback when no other API exists.

2. **Email: read directly from SQLite, NOT JXA.** macOS Mail stores all data in
   `~/Library/Mail/V*/MailData/Envelope Index` (SQLite). Query it directly with
   `sqlite3` or Swift's `SQLite3` framework. This is 10-100x faster than JXA.
   See `EmailReadSkill.swift` for the pattern.

3. **Calendar/Reminders: use EventKit, NOT JXA.** EventKit is the native Swift
   framework — fast, typed, no process spawning. Always prefer EventKit over
   JXA for calendar and reminders.

4. **Contacts: use Contacts framework, NOT JXA.** Swift Contacts framework is
   native and fast.

5. **Messages: SQLite, NOT JXA.** Messages stores chat.db in
   `~/Library/Messages/chat.db` (SQLite). Read directly. See `MessagesReadSkill.swift`.

6. **Notes: JXA is the ONLY option.** Apple Notes has no Swift framework and the
   SQLite store is private/encrypted. JXA via osascript is the only access path.
   Accept the performance cost. Bound the scan ceiling (max 100 notes) to limit
   exposure.

7. **Spotlight: use mdfind (Process), not manual file enumeration.** mdfind is
   indexed and fast. Manual FileManager enumeration is O(n) on all files.

8. **Obsidian/journal: FileManager direct read.** Markdown files on disk —
   read directly with FileManager. No JXA needed.

### Quick reference: data source → fastest access method

| Data | Fastest access | Why | Example skill |
|---|---|---|---|
| Email (Mail.app) | SQLite (`Envelope Index`) | 10-100x faster than JXA | EmailRead |
| Calendar | EventKit (Swift) | Native framework, no process spawn | CalendarRead |
| Contacts | Contacts framework (Swift) | Native framework | ContactsSearch |
| Messages | SQLite (`chat.db`) | Direct DB access | MessagesRead |
| Notes | JXA (osascript) | No framework, only option | NotesRead |
| Reminders | EventKit (Swift) | Native framework | (future) |
| Spotlight search | `mdfind` (Process) | Indexed search | SpotlightSearch |
| Obsidian/journal | FileManager | Plain markdown files | JournalRead |
| Finance | SQLite (hOS ledger) | Internal hOS DB | FinanceQuery |

## Past issues (avoid these)

1. **Force unwraps (`!`)** — use `guard let` / `if let` instead. Force unwraps crash on nil.
2. **Missing from SKILL_MANIFEST** — after creating a skill, add it to `scripts/package-release.sh`
   SKILL_MANIFEST array or it won't ship in releases.
3. **TCC description stale** — if adding a new Apple Events automation domain (Notes, Reminders),
   update the `NSAppleEventsUsageDescription` in `Info.plist`.
4. **Process deadlock** — when using `Process()` for JXA, always set `pipe.standardOutput`
   and read it. Not doing so can deadlock if output exceeds pipe buffer.
5. **Sort before limit** — always sort results BEFORE applying limit, not after.
6. **ADTools commands** — correct: `get-task`, `update-task`, `add-tag`, `remove-tag`.
   `show-task` does NOT exist.

## Code style

- Match existing skills (NotesRead, JournalRead, SpotlightSearch) for structure
- 150-250 lines is typical for a read skill
- Meaningful log messages at key points (not noisy, not silent)
- Comments only for non-obvious logic

## Process discoveries

7. **Previous agent did implementation but failed to commit** — always verify
   uncommitted work with `git status` before starting fresh. A prior agent run
   implemented Credential Vault + Error Logging (build passed) but exited
   without committing. The work was real and had to be recovered, reviewed, and
   committed — not re-implemented from scratch. Lesson: `git status` first,
   every time.
8. **Two features on one branch** — when two features land on the same branch
   and share files (ChatAdminViews.swift, ContentView.swift had interleaved
   changes for both Credential Vault and Error Logging), commit the shared
   files with the foundation feature. The second commit only needs the
   feature-exclusive new files (ErrorLogger.swift, ErrorLog.swift).
9. **status-ready-for-qa tag GID = 1217508257505680** — looked up via Asana
   tags API. status-in-progress = 1217507880143261. Add to your known tag GIDs.

## Feature branch workflow (2026-08-16)

The branch is `feature/{slug}` — already created by the requirements agent at
ready-to-plan, with specs committed on the branch in `docs/scope/{slug}.md`.

**When you start a build:**
- Read the branch name from Asana task notes (put there by requirements agent)
- The coordinator creates a worktree from the EXISTING `feature/{slug}` branch:
  `git worktree add /tmp/hos-build-{slug} feature/{slug}` — this is NOT a new
  branch, it's a checkout of the existing one
- Check out the existing `feature/{slug}` branch in the worktree
- Specs are already on the branch — read `docs/scope/{slug}.md` for the full spec
- Add your code on TOP of the specs already there

**All work stays on the same branch:**
- Code + QA notes + docs all committed to `feature/{slug}`
- One PR per feature — specs + code + QA notes + docs go to main together
- Do NOT create a separate branch for specs or docs
- Do NOT merge to main yourself — the coordinator opens the PR and merges

## Companion iOS App patterns (0.5.0)

10. **PBXFileSystemSynchronizedRootGroup** — the hOS (iOS) target uses
    Xcode 16+ "file system synchronized groups": any .swift file dropped
    into the `hOS/` directory is automatically included in the target. No
    pbxproj edits needed for iOS-side files. But the hOS Server (macOS) target
    uses explicit PBXBuildFile + PBXFileReference entries — you must manually
    add three sections to the pbxproj for each new server-side source file:
    PBXBuildFile, PBXFileReference, Sources build phase, and the main group's
    children list.

11. **CloudKit fixed-recordName pattern** — the entire iCloud communication
    (approvals + chat) uses index-free design: fetch records by FIXED, known
    recordName (not CKQuery). This avoids the CloudKit schema "recordName
    queryable index" problem that isn't auto-created and silently breaks
    queries in both Development and Production. Two records per channel:
    inbox (written by sender) + outbox (written by receiver). Single-writer
    per record → no save conflicts. Mirrored on both Mac and iOS sides.

12. **APNs graceful degradation** — APNsPushManager silently no-ops if the
    p8 key / team ID / key ID aren't configured via environment variables.
    The phone polls CloudKit every 4 seconds regardless. This means push is
    an optimization, not a dependency — the app works without Apple Developer
    APNs cert setup. When configured, push delivers realtime notifications
    for chat replies and approval requests.

13. **SwiftUI App lifecycle + APNs** — to register for remote notifications
    in a pure SwiftUI app (no UIApplicationDelegate), use
    `@UIApplicationDelegateAdaptor` to bridge the APNs delegate callbacks
    (didRegisterForRemoteNotificationsWithDeviceToken,
    didFailToRegisterForRemoteNotificationsWithError) back to the
    PushNotifications observable via NotificationCenter.

14. **xcodebuild destination warning is benign** — `xcodebuild: WARNING:
    Using the first of multiple matching destinations` appears because the
    project has both macOS and iOS targets. It does not affect the build.
    Use `-scheme "hOS Server"` to build the macOS server target.
