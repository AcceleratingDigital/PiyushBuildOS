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

## Doc update patterns (2026-08-18, stale-doc-cleanup)

15. **Cross-doc consistency on architectural decisions** — When a decision (like B2: SQLite→Postgres) revises earlier docs, check ALL references across decision docs, scope docs, and decision-derived content (D3's "Memory v1" sections). Use search + grep for the OLD term (e.g., "SQLite backup", "SQLite+NLEmbedding") to surface stale references. Patch all at once.
16. **API naming — check decision docs, implementation refs, and scope docs** — When standardizing API names (categorize→classify), search across ALL decision sections (B1 mentions in full capability registry, D1 mentions in shared capabilities, F8 mentions in cross-domain), skill-platform-runner scope (implementation+architecture), shared-capabilities scope (authoritative), and any "before build" docs. Decision docs can have 2-4 references per feature.
17. **Signature standardization in decision vs. spec docs** — Decision docs often write informal signatures ("emitKnowledge(entities:)") while spec docs have the formal 3-param version ("emitKnowledge(entities:, namespace:, provenance:)"). Find the spec doc as authoritative, then patch decision docs to match. Don't assume abbreviation in decisions is deliberate.
18. **Vapor/Network.framework ambiguity** — Avoid naming docs with implementation-choice terms ("vapor-gateway.md") when the decision later chose a different impl (Network.framework). Either rename the file + update refs, or annotate the doc with a clarification header noting the discrepancy. Postgres-related docs that cite the gateway should not claim "Matches D3's Vapor gateway choice" — check the actual gateway.md decision first.
19. **Superseded decision markers** — When a later decision (B2) revises an earlier one (F4), mark F4 explicitly as "SUPERSEDED BY B2" with the date. Don't leave ambiguous references like "NOT in 0.9 scope" when 0.9 scope has now changed. Link explicitly to the revised decision.
20. **Model names and runtime config in specs** — When a spec lists example model names ("nomic-embed-text", "gemma2:2b"), add a NOTE explicitly stating which fields are examples (model names, maybe provider URLs) vs. architecturally binding (dimensions, context windows). Runtime config that can override names without recompile must be documented clearly to avoid confusion with architectural requirements.

21. **Comprehensive grep before claiming standardization complete** — When fixing API naming across multiple docs (e.g., categorize→classify), do not rely on a task spec that says "3 occurrences in file X." Audit all occurrences first: `grep -r "categorize" docs/` (or the target file). The actual count may be 8 total with only 4 already fixed by a prior agent, leaving 4 more to standardize. Use grep early, verify the complete set, and ensure the replacement is truly exhaustive across all relevant files. Decision docs often have multiple references per feature (B1, D1, F8 sections), all needing updates when the API name changes. Incomplete replacements create contradictions between decision docs and implementation specs that confuse future readers about which name is correct.

22. **Rebase reconciliation: incompatible role models** — When a feature branch uses a different enum (e.g., MemberRole with owner/admin/member/guest) than main (owner/parent/child), you cannot merge the branch's UI code. Main's model with MemberAccessController + MemberPermission is canonical. Port only model-independent utilities (like availableMacUsers via dscacheutil) as nonisolated static funcs on the existing service. Do not port structs/enums that conflict with main's model.

23. **Rebase old branch: resolve by taking main's version for ALL conflicting files** — When a branch is based on old main and main has since shipped the same feature differently, `git checkout main -- <path>` for every conflicted file. The branch's diff against old main shows "deletions" of files that main added later — those are not real deletions, just age artifacts. Keep main's version for unrelated files (site/*, docs/*, scripts/*) without exception.

24. **Shared file guard during rebase** — If a rebased commit adds a file under site/ (shared file), remove it with `git rm` even if it's a new file not on main. The "do not modify shared files" rule applies to additions too. Commit the removal separately with a clear message.

25. **Skill bundles compile separately from the Xcode project** — Skills in `skills/*/` are NOT in the pbxproj; they're compiled by `scripts/build-skill.sh` using `swiftc` directly, linking only SkillKit.framework. The `xcodebuild` build only compiles the app target + SkillKit. To verify a skill compiles, run `swiftc` manually against the built SkillKit.framework. Skills can use Foundation/SQLite3 directly (like NotesRead uses Process), but cannot access internal app types like SystemSQLiteReader (not public) — import SQLite3 and inline the copy-then-read pattern instead.

26. **nonisolated static methods for Task.detached in @MainActor skills** — Skills are `@MainActor` (Skill protocol requires it), but file I/O in `Task.detached` runs off-actor. Static helper methods called from detached context must be marked `nonisolated` or the compiler rejects the call. This applies to any pure-function helpers (timestamp conversion, DB queries) invoked from the detached task.

27. **PBXFileSystemSynchronizedRootGroup auto-discovers new files** — SkillKit uses PBXFileSystemSynchronizedRootGroup in project.pbxproj, so adding a new .swift file to the SkillKit/ directory requires NO pbxproj edit. It is automatically included in the build. Only files at the project root level (like PolicyCheckpoint.swift) need explicit PBXFileReference + Sources build phase entries.

28. **Rebase with type collisions: add convenience APIs, don't delete either side** — When a branch and main both define the same types (e.g., KnowledgeEntity, ClassificationRule) with different shapes, don't pick one and rewrite all call sites. Instead, keep main's richer types and add convenience initializers + computed properties that accept the branch's calling conventions. This preserves all existing code (concrete classes, shipped skills) while supporting the branch's retrofitted skills without modifying them.

29. **Preserve existing output behavior when adding LLM enhancements** — When retrofitting a skill to use shared capabilities (classify, summarize, emitKnowledge), the existing `SkillResult.summary` output format MUST be preserved as the primary summary. The old code returned structured digests (sender frequency tables, per-member briefings) that downstream consumers and the LLM loop depend on. Replacing them with an LLM-generated summary breaks behavior in two ways: (a) when the LLM is available, the structured format is lost; (b) when the LLM is unavailable, the fallback is `String(content.prefix(N))` — a truncation that is strictly worse than the old full output. Correct pattern: keep the structured digest/briefing as `summary`, store the LLM summary in a separate `fields` key (e.g., `digest_summary`, `briefing_summary`), and only store it if the provider genuinely synthesized something shorter (not a prefix truncation). This way the LLM enhances but never degrades.

30. **Shared capabilities with LLM fallback must not be called in per-item loops** — `context.classify()` does rule-matching first (fast) but falls through to `llm.quickComplete()` for unmatched content. Calling it in a per-header loop over 100 mail headers triggers 80-90+ sequential LLM calls. The pre-retrofit code used a simple `actionWords.contains` filter — O(n) string matching, zero LLM calls. Correct retrofit pattern: keep the fast heuristic as pass 1 (filter), then call classify() only on the small subset that matched (pass 2, refinement). This preserves O(n) for the common case and limits LLM calls to confirmed matches. Same principle applies to any shared capability with LLM fallback: never call it in a loop over a large collection without a pre-filter.
- 2026-08-19: When consolidating N separate UPDATEs into a single SQL UPDATE with PostgresNIO, use PostgresQuery(unsafeSQL:sql, binds:binds) with $N placeholders — column names are hardcoded (not user-supplied) so unsafeSQL is safe; only values go through PostgresBindings. This is the same pattern used in MemoryStore/MemoryQueryBuilder.
- 2026-08-19: For path-parameter routes in CompanionServer (e.g. DELETE /loops/:id), handle them BEFORE the exact-match switch statement using hasPrefix — the switch only matches (method, exactPath) pairs. The /mcp/* passthrough already uses this pattern.
- 2026-08-19: When changing a method return type from Bool to an enum (e.g. requestExecutionApproval Bool→ExecutionApprovalResult), check for callers first — if the only reference is the definition (agent-side callers not in the codebase), the change is safe. Callers in the agent layer will need updating when they're integrated.
- 2026-08-19: OpenLoopStore.update() consolidated from 6 separate UPDATE queries (worst case) to 1 — only includes changed fields via optional/nil pattern. Uses dynamic SET clause building with PostgresBindings.
- 2026-08-19: B1 brief push — when B4 CKSubscription push infra is already shipped, verify what exists before building. The brief-outbox writeBriefOutbox(), CKQuerySubscription, and deep-link routing were ALL already in place. The actual missing pieces were: (1) LLM-generated previewSummary (not prefix truncation), (2) per-member brief time config (default 7 AM, was 6 AM), (3) RootView missing .onReceive for .hosNavigateToToday (push posted the notification but no tab switch happened), (4) BriefPanel not refreshing on deep-link. Always grep for existing infrastructure before assuming it needs to be built.
- 2026-08-19: Privacy badge (U8) — for additive UI+metadata features, placing new Swift files in the PBXFileSystemSynchronizedRootGroup directories (hOS Server/ for Mac, hOS/ for iOS) avoids any pbxproj edits; files are auto-included. When the iOS and Mac targets need the same component name but different behavior (e.g. PrivacyBadge), write separate files per-directory — they compile into separate targets so there's no conflict. Attach metadata to ChatEntry via an optional field with a default-nil init so all existing call sites compile unchanged.
