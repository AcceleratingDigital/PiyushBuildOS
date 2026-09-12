# Coder Agent — Learned Context

> **SYNC NOTE:** This file is shared between all surfaces that run coder agents.
> Update it at every significant event so future runs (cron or interactive) stay aligned.
> Learnings that affect OTHER roles (QA, review, build) go to SHARED-CONTEXT.md, NOT just here.

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
> **READ FIRST:** `SHARED-CONTEXT.md` — shared context for ALL agents.
> Read it at session start before this file. It contains project identity,
> S-S-D model, communication channels, repo layout, Asana tags, tool/model
> matrix, release pipeline, concurrency guardrails, and known issues.
> Update it when shared state changes; keep role-specific instructions here.



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
- 2026-08-20: QA rework (O2 grocery list) — when `import os` and `import Logging` coexist, `Logger` is ambiguous; qualify as `os.Logger` vs `Logging.Logger`. Actor-isolated stores using PostgresNIO must use `nonisolated(unsafe)` for module-level `Logging.Logger` instances. When converting a `@MainActor` store to an `actor`, all call sites need `await`. The iOS target (hOS/) doesn't include root-level server files — duplicate any shared enums (e.g. GroceryCategory) in the iOS view model file.
- 2026-08-20: O7 news digest — XMLParserDelegate.parser(_:foundCDATA:) takes `Data` not `String`; using String causes a "nearly matches optional requirement" warning and the delegate method is never called. Skill bundles with only skill files (no app-layer shared code) don't need pbxproj entries — build-skill.sh compiles them via swiftc directly.
- 2026-08-20: O8 weather-aware suggestions — skill bundles are separate swiftc compilation units and CANNOT import sibling skill bundles (e.g. WeatherAwareSuggestions can't `import WeatherRead` to reuse WeatherService/WeatherConfig/WeatherLocation). Each skill must inline its own fetch + define its own types. When one skill wants to inherit another's persisted config, read the KB namespace raw JSON via JSONSerialization (not by decoding to the sibling's Codable type) and extract the fields by hand. New brief data-source tools in LLMService: add to briefTools array, the dispatcher condition (~line 1705 `rawName == ...`), and handleBriefTool's switch; the tool can invoke a loaded skill via host.runReturningOutcome(skillID, arguments:) and degrade gracefully when the skill isn't loaded. SuggestionThresholds (or any config struct with Set<Int> fields) needs an explicit Codable impl with decodeIfPresent so partial configs merge with defaults. Array.sorted with a multi-statement closure needs explicit type annotations `(a: T, b: T) -> Bool in` or the compiler picks the SortComparator `sorted(using:)` overload and fails to infer.
- Kid Surface (B5): LLMService.reply() must thread memberRole so refreshSystemMessage+autoRecall can skip owner-scoped notes/directives/memories for .child role — otherwise kids see the adult prompt.
- 2026-08-20: D2 meal plan loop — skill bundles cannot reference app-target types (MealStore/PantryStore/GroceryStore), so all meal/pantry data flows through the HostContext broker seam as JSON: pantryItems(), saveMealPlan(planJSON:), groceryAddItems(itemsJSON:), currentMealPlan(), mealBrief(). The server side (PolicyCheckpoint) parses the JSON into MealPlan/MealEntry/Ingredient and persists to Postgres. Pattern for a new data vertical: (1) add SkillBroker protocol methods + HostContext conveniences in SkillKit.swift, (2) implement on PolicyCheckpoint with .household/.read for reads and .household/.mutate + requireApproval for writes, (3) wire into the skill via context.<method>(), (4) for the Daily Brief add a tool to LLMService.briefTools + a handleBriefTool case calling checkpoint.<seam>() (returns JSON; the agent synthesizes) AND optionally a @MainActor BriefProvider struct in hOS Server/ for natural-language formatting. .planning is NOT a Capability.Domain — use .household for meal/pantry/grocery. HostContext.kbRead/kbWrite are `async` — callers need `await` (the standalone swiftc build catches this even when the app build doesn't because of @MainActor inference differences). nonisolated static lets on constant data (e.g. meal template arrays) so a nested struct's instance method can reference Self.<static> without main-actor isolation errors. New files under hOS Server/ are auto-included via PBXFileSystemSynchronizedRootGroup (no pbxproj edit); root-level .swift files (MealStore.swift, PantryStore.swift) still need explicit 4-section pbxproj entries. build-skill.sh: `CONFIGURATION=Release PRODUCTS_DIR=<derivedData>/Build/Products/Release bash scripts/build-skill.sh <Dir> <Bundle> <Class> <id> <ver> --no-deploy` — always verify the bundle compiles standalone after touching a skill.
- 2026-08-20: O5 package tracking — name-collision pitfall: an existing `struct Package` (admin subsystem-toggle in `hOS Server/AdminPackageConfig.swift`) collides with a new package-domain `Package` model. Rename the domain model (`TrackedPackage`) — `\bPackage\b` perl rename is safe because compound tokens (`PackageStore`/`PackageStatus`) don't match the word-boundary pattern. When a data vertical's model name could clash, grep `struct <Name>` across the whole server tree before committing to it.
- 2026-08-20: O5 — `Calendar.isDate(_:inSameDayAs:)` and date comparisons take `Date`, NOT `Double` epoch. If your model stores timestamps as `Double` (timeIntervalSince1970, matching GroceryStore/MealStore convention), wrap every calendar/comparison call site in `Date(timeIntervalSince1970:)` — the compiler error "cannot convert Double to Date" is the signal. Don't store `Date?` in Postgres-backed Codable models; keep `Double?` and convert at the call site (consistent with the rest of the codebase).
- 2026-08-20: O5 — PostgresNIO `PostgresQuery(stringInterpolation:)` renders `Double?` nil as NULL directly (no `PostgresQuery.Null`/`PostgresQuery.Double` API exists — those are invented). Mirror OpenLoopStore: interpolate the optional `Double?` straight into the query. For conditional null writes (e.g. "set actual_delivery only if delivered"), bind to a `let x: Double? = cond ? now : nil` first, then interpolate.
- 2026-08-20: O5 — follow the D2 MealPlanning broker-seam pattern for new data verticals, NOT the GroceryList direct-reference pattern. GroceryListSkill references `GroceryStore()` directly, which does NOT compile as a standalone skill bundle (coder.md lesson 30 confirms skill bundles can't import app-target types). The canonical pattern: (1) actor store in app target with `packagesAsDictionaries()`/JSON helpers, (2) broker protocol methods + HostContext conveniences in SkillKit, (3) PolicyCheckpoint impl with requireCapability + requireApproval for writes, (4) skill builds JSON and hands it to `context.<seam>()`, (5) `@MainActor` BriefProvider in `hOS Server/` for natural-language formatting + a brief tool in LLMService (briefTools array + dispatcher `rawName ==` condition + handleBriefTool switch case). The brief tool's skillID is the new skill's manifest id (it must declare `.household.read`).
- 2026-08-20: O5 — schema bootstrap is central via `PostgresSchema.bootstrap()` (not per-store `bootstrapSchema`). Add the new table's `static let <name>Schema` to PostgresSchema.swift AND a `for stmt in splitStatements(<name>Schema)` loop in `bootstrap(logger:)`. Stores lazy-connect; the schema must exist before first access.
- 2026-08-20: O5 — mail seam returns headers only (subject + sender, NO body) via `HostContext.searchMail`/`recentMail`. For any feature needing email-body content (tracking-number extraction, etc.), detection must key off sender domain + subject, with the shared-capability `extract(content:schema:)` as an LLM fallback. `ExtractionSchema.Field` has TWO inits: `init(name:description:required:)` (canonical) and `init(name:hint:)` (convenience, no required). The standalone `swiftc` skill build rejects the `hint:` label when combined with `required:` — use `description:` consistently in skill bundles.
- 2026-08-20: O5 build infra — a killed xcodebuild leaves the XCBuild `build.db` locked; the next build fails with "database is locked" / "disk I/O error", NOT a code error. Recovery: `pkill -f XCBuild`, wipe DerivedData, rebuild. The `xcodebuild: error: Could not resolve package dependencies` that follows the lock error is a symptom, not a cause. Always check for a lingering xcodebuild process (PID) before treating these as code failures.
- 2026-08-20: U5 rebase — when both main and feature branch add new methods/sections to the same area of a Swift file, the conflict spans the entire block. Resolution: keep main's block, close it with the proper `}`, then add the feature's new MARK section + method after. The `<<<<<<<`/`=======`/`>>>>>>>` markers often sit between two sibling blocks that should both remain — the feature side is NOT a replacement for main, it's an addition.
- 2026-08-20: COORDINATION.md log conflicts during rebase are append-only — both sides add entries at the end. Remove the conflict markers and keep BOTH sets of log entries (main's + feature's). Do not delete either side's log entries.
- 2026-08-20: O6 birthday rework — CNLabeledValue<DateComponents>.value bridges as NSDateComponents (not DateComponents) through the Obj-C Contacts API; `labeledValue.value as NSDateComponents` then copy fields into a `var DateComponents()` (must be `var` — `let` rejects property assignment). CNContactDatesKey + CNLabelDateAnniversary are the standard Contacts framework symbols for anniversary dates. Custom CN labels look like `_$!<Label>!$_` — fall back to a generic label for readability.
- 2026-08-20: O6 — ContactCard (from searchContacts) does NOT carry the CNContact identifier, so a skill can't get a contactId for gift recording via searchContacts. Resolve the stored birthday entry by name via birthdayEntries() instead — the entry's contactId is the CNContact identifier the store dedups on. Always check what fields a broker return type actually exposes before relying on one.
- 2026-08-20: O6 — per-member scoping for a new data vertical follows the same pattern as PackageStore/OpenLoopStore: add a `member_scope` column, filter every query by it, and have PolicyCheckpoint pass `who.isOwner ? "household" : who.id` as the scope. The owner sees the whole family's data (intended); a non-owner is scoped to their own id. Add a `birthdayScope(for:)`-style helper on the checkpoint mirroring `financeScopes`/`alsohouseholdScope`.
- 2026-08-20: O6 — lead-time push reminders reuse the B4 CKSubscription path, NOT a new APNs p8 flow: add a `writeBirthdayReminderOutbox` (type="birthday", unique-per-event recordID) on CloudMailbox, register an iOS CKQuerySubscription with `predicate: type == "birthday"`, and run the daily dedup check in SkillScheduler as a special-cased non-skill entry (like chore-regeneration/nightly-backup) that reads the store + pushes directly. Dedup via a `birthday_reminders` table keyed (entry_id, lead_time_days, reminder_year) so each lead time pushes once per birthday year.
- 2026-08-20: O6 — a skill must thread `arguments["member"]` (default nil/owner) through EVERY broker call, not just some. member=nil → resolve() defaults to owner → who.isOwner always true → privilege checks are bypassed. This is the privilege-escalation bug pattern for any owner-gated broker method; the skill, not the broker, owns passing the real member.
- 2026-08-20: O6 — wire a @MainActor BriefProvider (natural-language formatter) into the Daily Brief by late-injecting the store into LLMService and calling the provider in handleBriefTool's case, returning `{"summary": <escaped NL>, "data": <raw JSON>}`. The provider complements (not replaces) the checkpoint JSON seam — keep both so non-LLM consumers keep the structured path.
- 2026-08-20: D6 — wrap multi-step Postgres operations (status UPDATE + star deduction) in an explicit BEGIN/COMMIT transaction on the cached _conn; catch any error (GamificationError or otherwise) to ROLLBACK before rethrowing, ensuring the two writes are atomic and failures can't leave the DB in a half-applied state.
- 2026-08-20: U3-U9 — when multiple UX items overlap with already-shipped files (e.g. SuggestedPromptsView, PrivacyBadge, ActivitySummaryCard), skip the new-file step and just wire integration points; the build gate catches any mismatch between what you think is already there and what actually is.
- 2026-08-20: D1 finance security — PostgresQuery(unsafeSQL:) + hand-rolled escapeSQL (doubling single quotes) on DML is a SQL-injection risk on user-controlled fields; never use it for INSERT/UPDATE/DELETE with variable values. Convert to parameterized PostgresQuery("...") with \(variable) interpolation — PostgresNIO binds each interpolated value as a separate parameter, so strings/UUIDs/Doubles/optionals all bind safely with no manual quoting. For an optional Double? (e.g. approved_at) interpolate \(theOptional) directly and nil renders as NULL (no .map{String($0)} ?? "NULL" needed). Schema-bootstrap DDL and constant SELECTs (no user input) may keep unsafeSQL since there is nothing to inject.
- 2026-08-20: Skills in skills/*/ are NOT compiled by `xcodebuild` (only the app target + SkillKit are) — to verify a new/modified skill actually compiles, run swiftc directly against the built SkillKit.framework: `xcrun swiftc -parse-as-library -emit-library -module-name <Name> -target arm64-apple-macos26.5 -sdk "$(xcrun --show-sdk-path --sdk macosx)" -F "<PRODUCTS_DIR>" -o /tmp/<Name>.dylib skills/<Dir>/*.swift`. A passing xcodebuild does NOT mean the skill compiles. Also: the read-side contacts skill is `ContactsSearchSkill` (there is no "ContactsRead"), and `ContactCard` (SkillKit) intentionally does NOT carry the note field — relationship inference from notes needs a future notes-capable read API; design the graph so it degrades gracefully with nil notes (role queries + LLM classify still work).
- 2026-08-20: U2 Family Dashboard — LLMService.host is private; use @Environment(SkillHost.self) directly to run skills from views. LLMService.loopStore is a `let` (accessible) but LLMService.finance is optional and accessed via llm.finance (not checkpoint). FinanceStore.CategorySpend uses totalCents (Int), not amount (Double) — divide by 100.0 for display. OpenLoop.id is UUID (not optional) — use loop.id.uuidString directly. On macOS server, MemberService has no "selected member" concept (always owner context); member-aware card gating is iOS-only via MemberStore.isSelectedChild. DashboardCardType enum defined in both hOS Server/ and hOS/ targets compiles fine — separate targets, no conflict.
- 2026-08-20: U2 rework — PolicyCheckpoint.requireCapability() checks manifests[skillID] — a bogus skillID like "dashboard" throws capabilityNotDeclared silently (caught → empty card). Always grep SkillScheduler.swift + skill manifests for the real registered skill ID before calling checkpoint methods. calendarToday returns "start" as a formatted String ("9:00 AM"), not a Double epoch — parse as String and use directly. packageBrief() returns keys arrivingToday/arrivingTomorrow/thisWeek/inTransitCount, not "packages". Split independent API calls into separate do/catch blocks so one failure doesn't blank the entire card.

## D4 Knowledge Query session (2026-08-20)
- Skill bundles cannot import sibling skill types — use [[String:String]] adapter dicts for cross-skill normalization, not the normalized struct type
- Composite skills that need to query sibling skills must reimplement the source logic internally (can't call sibling skill bundles at runtime per lesson 30)
- Journal vs Obsidian routing: YYYY-MM-DD filenames / "journal" folder / "journal" tag → journal; everything else → obsidian (avoids duplication)
- nonisolated static methods for vault helpers called from Task.detached context (lesson 26 pattern)
- Coder subagent (max_iterations 50) is insufficient for M-scope features with LLMService wiring — coordinator must finish: brief tool handler case + SKILL.md + build + commit
- 2026-08-21: When moving a SwiftUI view to a different tab/surface, also remove its now-unused @Environment properties from the old host — Swift will warn (or error) on unused @Environment vars in the observable macro world. Always grep for remaining usage before assuming an @Environment can stay.

- 2026-08-22: EKEventStore authorization for .event must check BOTH .fullAccess AND .authorized — the reminders provider already did this (DataProviders.checkAuth ~line 780) but CalendarProvider only checked .fullAccess, so macOS 26 / legacy installs where TCC reports .authorized were rejected as notAuthorized. Mirror the reminders pattern anywhere EventKit auth is gated. Also: defaultCalendarForNewEvents can be nil (fresh install / disabled account) — fall back to store.calendars(for:.event).first{allowsContentModifications} before save() or it throws. And surface EventKit save errors by wrapping in SkillError.writeNotSupported(errorDescription) — a bare throw lets the LLM see only a generic Error and report "technical error" while the real cause is in os_log.
- 2026-08-22: Audit-after-write ordering matters: PolicyCheckpoint does createEvent() THEN auditAllow(). If Postgres is down, auditAllow throws .auditWriteFailed (fail-close) AFTER the event already landed in Calendar.app — rethrowing it tells the LLM "access denied" and hides the created event. When a side-effect precedes the audit, catch audit failure separately and surface "event created but audit failed" (writeNotSupported) so the person knows the event landed and won't re-create a duplicate. Pattern applies to every mutate path that writes before auditing (sendMail, reminders, contacts). (docs: coordination log + coder lessons for calendar create-fails fix)
- 2026-08-22: When hoisting an @Observable from local @State to @Environment, the pattern is: (1) create instance in App.init, (2) inject via .environment() in body, (3) change consumer from @State to @Environment(Type.self), (4) update PreviewHost to also inject. The @Bindable pattern still works for child views that receive the manager as a parameter — only the top-level owner changes. PackageConfigManager was already @Observable/@MainActor, so no conformance changes needed.
- 2026-08-22: The spec referenced "FamilyDashboardView" which doesn't exist in the codebase — the macOS Server uses DashboardView (in hOS Server/ContentView.swift) and the iOS companion uses TodayView (hOS/TodayView.swift). Always grep for actual file names before starting work; scope docs may reference planned-but-unimplemented names.
- 2026-08-22: Task prompts may carry stale/invalid Asana status tag GIDs — Asana returns "Unknown object" for them. Trust the verified canonical GIDs in agent memory (status-in-progress=1217507880143261, status-ready-for-qa=1217508257505680); run `list-task-tags` first to see what's actually applied before assuming a remove is needed. Also: Member.swift uses a custom Codable impl (not synthesis) — adding a new optional field requires adding it to the CodingKeys enum AND both decodeIfPresent (init(from:)) and encodeIfPresent (encode(to:)), or it silently round-trips to nil on write even if set in memory.

## CKShare Phase 0 (2026-08-22)

31. **CloudKit async API names differ from naive guesses** — `CKDatabase` async methods don't always match the ObjC name. The no-arg "fetch all zones" is `db.allRecordZones()` (NOT `db.recordZones()`, which returns a `[CKRecordZone.ID : Result<CKRecordZone, Error>]` dict for the per-ID fetch). Delete zone is `db.deleteRecordZone(withID:)` (NOT `withZoneID:`). Always grep the SDK headers (`CloudKit.framework/Headers/CKDatabase.h`, `CKError.h`) for the `NS_SWIFT_ASYNC_NAME` to confirm the Swift label before committing.
32. **CKError code matching** — `CKError` has no top-level `.recordZoneNotFound` case. Match via `catch let error as CKError where error.code == .zoneNotFound` (the code lives on `CKError.Code`, value 26). Same pattern for any CKError partial-function catch: bind the error and compare `.code`, don't try to use a non-existent enum shorthand.
33. **Best-effort startup provisioning pattern** — for idempotent CloudKit zone/record provisioning at app launch, expose two entry points: the throwing `provisionZones(members:)` for test/explicit callers, and a non-throwing `provisionZonesOnStartup(members:)` wrapper that logs+swallows errors. Call the latter from a fire-and-forget `Task` in `hOS_ServerApp.init()` after MemberService loads. CloudKit outage must never block server startup — zones retry next launch.
- 2026-08-22: CKModifyRecordZonesOperation.perRecordZoneSaveBlock signature on macOS 26 SDK is `((CKRecordZone.ID, Result<CKRecordZone, Error>) -> Void)?` — NOT the older `(CKRecordZone, Error?)`. The first param is a zone ID (use `.zoneName` directly, no `.zoneID`), and the second is a Result you switch on (.success/.failure), not an optional Error. Don't assume the legacy ObjC-bridged signature; let the compiler guide and switch on Result.
- 2026-08-22: When adding a health check to HOSDoctor that depends on a list only available via @Environment (MemberService), add a `runFullCheck(members:)` overload and keep the nil-arg `runFullCheck()` delegating to it. Existing call sites that don't have members (RestoreEngine.runAllGates, DiagnosticBundleExporter.exportBundle) keep working via the nil path which skips the check; only the DiagnosticsPanel (which has @Environment(MemberService.self)) passes members. This avoids threading MemberService through every doctor caller.
- 2026-08-22: HOSDoctor.swift references CKRecordZone properties ($0.zoneID.zoneName from ZoneManager.fetchAllZones()) — needs `import CloudKit` even though it only touches ZoneManager's return type, because the CK property accessors live in the CloudKit module.
- 2026-08-22: Vault Add Entry LLM-only fix — when generalizing a hardcoded credential model, keep the existing explicit struct fields for the original type (llmProvider) and add a flexible `[String: String]` payload (`fields`) for new types rather than an enum-with-associated-values. The dedicated LLM fields are still read by the LLM read path (loadEntry reads individual keychain keys, not the blob) AND by named-entry blobs already on disk, so they must stay. Backward-compat decoder: use `decodeIfPresent` for the new `type`/`fields` keys AND loosen the previously-strict `decode(String.self)` for providerURL/apiKey to `decodeIfPresent` so old partial blobs don't throw. Existing blobs without `type` default to `.llmProvider`.
- 2026-08-22: Type-driven SwiftUI form pattern — drive field rendering off a `CredentialType.fieldSchema(for:)` static func returning `[CredentialFieldSpec]` (key/label/required/secret/placeholder), then `switch state.credentialType` in the form body to pick llmFields / typedFields / customFields. Per-field secret visibility toggles need a `@State private var secretVisible: [String: Bool]` dict (not a Bool per field) since the typed fields are data-driven. On type change, re-seed the `fields` map to the new schema (`onChange(of: state.credentialType)`) so bindings don't reference stale keys.
- 2026-08-22: `testConnection` returning nil-on-success is a footgun for the UI — the form's `if let result = testResult` only fires on non-nil (failure/status), so the green "Connected" branch (`result == "Connected" || result.contains("success")`) never triggers. Preserve existing behavior unless the task scopes the fix; flag it as a v2.

- 2026-08-22: SwiftUI Picker selection tags require Hashable conformance. AvailableUser was Identifiable+Sendable but not Hashable — Picker/Optional.tag() rejected it with "requires that 'AvailableUser' conform to 'Hashable'". Adding `Hashable` to the struct protocol list is the minimal fix (UUID + two Strings auto-synthesize). When a spec says "no changes needed to file X" but the UI integration requires a protocol conformance, the minimal additive change (adding a protocol conformance) is acceptable and doesn't break existing callers.
- 2026-08-22: Accessibility retrofit pattern — when adding accessibilityLabel/accessibilityHint to TextFields in a SwiftUI form, chain them directly after .textFieldStyle(.roundedBorder). Retrofit adjacent fields in the same form section, not just the field named in the spec. Placeholder text should never contain literal asterisks — they read as masked/redacted data to users and screen readers.

- 2026-08-22: CloudKit environment (Development vs Production) is NOT readable via a public runtime API — it's bound by the app's provisioning profile. The best runtime proxy is the APNs environment entitlement (`aps-environment` / `com.apple.developer.aps-environment`: `development`↔Dev, `production`↔Prod), read from the code signature via SecStaticCodeCreateWithPath + SecCodeCopySigningInformation(kSecCSSigningInformation) → kSecCodeInfoEntitlementsDict. CRITICAL: those code-signing query symbols are macOS-ONLY — `import Security` alone does NOT bring them into scope on iOS (compiler errors "cannot find type 'SecStaticCode'"). Guard the entire body with `#if os(macOS)` (NOT `canImport(Security)`, which is true on iOS but the symbols still aren't exported); iOS falls back to "unknown". For "approvals not syncing" diagnostics the environment matters most on the Mac (writer) side; on iOS the account-status check is the key signal.
- 2026-08-22: CKError structured logging — `CKError.Code` has NO `errorDescription` property (it's not like CFError's localizedDescription). Use `String(describing: ckErr.code)` for the enum-case name, alongside `ckErr.code.rawValue` for the numeric code. Pattern: `catch let ckErr as CKError { log("code=\(ckErr.code.rawValue) (\(String(describing: ckErr.code))) — \(ckErr.localizedDescription)") }`. Keep the `.unknownItem` / `.serverRecordChanged` partial-function catches BEFORE the generic `catch let ckErr as CKError` so the handled cases don't fall through to the error log.
- 2026-08-22: Surfacing @Observable diagnostics from a long-running service into SwiftUI — convert the service (CloudMailbox) to @Observable, add a `private(set) var diagnostics = SomeDiagnosticsStruct()` updated each sync cycle, inject the service instance into the environment via `.environment(cloudMailbox)` in the App body, and read it in the view with `@Environment(CloudMailbox.self)`. Use `@Bindable var mailbox` when the view needs to call mutating methods. The struct value-type diagnostics auto-trigger view updates on reassignment.
- 2026-08-22: HOSDoctor overload pattern for new optional dependencies — keep the existing no-arg `runFullCheck()` → `runFullCheck(members:)` → add `runFullCheck(members:cloudMailbox:)` chain. Each level delegates down with nil for the deps it doesn't have. Existing callers (RestoreEngine.runAllGates via no-arg, DiagnosticBundleExporter.exportBundle via no-arg) keep working unchanged; only the DiagnosticsPanel (which has @Environment(CloudMailbox.self)) passes the mailbox. Don't thread the dep through every doctor caller.
- 2026-08-22 (review rework): **Fail-secure error handling in async retry paths** — `_ = try? await db.save(server)` in a conflict-retry branch is a silent-success footgun: it swallows the retry failure and returns `true`, so the caller stamps `lastOutboxWrite` and reports healthy for a write that never landed. Capture the retry result with a nested `do/catch` and return `.failure` (with the CKError code) on throw. Generalized lesson: any `try?` on the operation whose outcome determines a health/success signal must be replaced with a real `try` + error capture.
- 2026-08-22 (review rework): **Tri-state fetch beats nil-collapsing for health surfaces** — returning `nil` from `fetchRecord` for BOTH `.unknownItem` (record absent — normal) AND any other CKError (network/quota/auth) makes a CloudKit outage indistinguishable from "no decisions yet," and prevents `lastInboxFetchError` from ever being set. Fix: return an enum `FetchResult { case found(CKRecord); case absent; case error(String) }`. `.absent` clears the prior fetch error (read path works); `.error` sets it with the CKError code. Callers switch on the enum instead of `if let`. Apply the same pattern anywhere nil is overloaded to mean two different things.
- 2026-08-22 (review rework): **Health status must aggregate ALL signals, not just the write path** — a CloudSyncStatusCard that keys glyph/color/pill off only `lastOutboxWriteError` reads green while the inbox fetch is broken or the iCloud account is unavailable. Factor in outbox write + inbox fetch + account status; any unhealthy → warning/error. Introduce a `HealthIssue` severity enum (none/idle/warning/error) computed from the three booleans so glyph, color, statusLine, and pill all derive from one source of truth. `accountStatus != "available"` is an error (sync cannot run), not neutral.
- 2026-08-22 (review rework): **Use structured status enums for UI coloring, not string contains** — `result.contains("OK")` for test-result coloring is fragile (a future "Sync OK but…" or localized string breaks it). Return a `TestSyncResult { let status: OutboxHealthResult.Status; let message: String }` and switch on `.ok/.warn/.fail` for glyph + color. Keep the human-readable message for display, but never parse it for control flow.
- 2026-08-22 (review rework): **Surface CKError codes in the admin UI, don't punt to "see logs"** — `writeRecord` returning `Bool` throws away the error code the caller needs. Return a `WriteResult { case success; case failure(String) }` where the failure string includes `CKError \(code.rawValue) \(String(describing: code))`. The diagnostics struct's `lastOutboxWriteError`/`lastInboxFetchError` then carry the code into the Admin → Health detail rows. Same for `checkOutboxHealth`'s `.fail` detail.
- 2026-08-22 (review rework): **`defer` for async probe cleanup** — a `testSync` probe that writes then reads then deletes can leak the probe record if readback fails and returns early. `defer` can't `await`, so gate the async `deleteRecord` in a `Task { _ = await deleteRecord(...) }` inside `defer`, flagged on a `var didWrite` set true only after the write succeeds (don't delete a record we never created). Pattern: `var didWrite = false; defer { if didWrite { Task { await cleanup() } } }`.
- 2026-08-22 (review rework): **a11y on diagnostic cards** — add `.accessibilityElement(children: .combine)` + `.accessibilityLabel` to detail rows and the status header so VoiceOver reads "iCloud account: available" rather than disjoint fragments. Decorative glyphs get `.accessibilityHidden(true)`. Buttons get `.accessibilityHint`. The combined header gets an `.accessibilityValue` summarizing the detail rows. This is a repeat-flagged pattern — retrofit when touching any health/status card.
- 2026-08-22 (review rework): **Frame developer tokens with context, don't show raw** — a raw container ID like `iCloud.AcceleratingDIgital.hOS` in a detail row labeled "Container" is an opaque token to an admin. Label it "iCloud Container" so the row reads as "iCloud Container: iCloud.AcceleratingDIgital.hOS". Minor copy framing, but it's the difference between actionable and inscrutable.

- 2026-08-23 (CKShare Phase 2): **CKRecordZone.defaultName doesn't exist in Swift CloudKit — use the literal string "_default" for the default zone ID.** When constructing a default-zone CKRecordZone.ID for migration fetches, CKRecordZone.ID(zoneName: "_default", ownerName: CKCurrentUserDefaultName) is the correct API.
- 2026-08-26: FeatureFlagManager (superadmin-panel-beta) — the hOS Server directory is a PBXFileSystemSynchronizedRootGroup, so new .swift files placed in `hOS Server/` are auto-included in the macOS target build (no pbxproj edits needed). The existing `SkillHost.disabledPackageIDs()` is `nonisolated static`, so to read FeatureFlagManager's cache from it, read the cache file directly via FileManager+JSONDecoder (not through the @MainActor manager instance). Pattern: nonisolated code reads the cache JSON file, @MainActor code uses the observable manager.
- 2026-08-26: When adding a new observable service to hOS_ServerApp, follow the existing pattern: (1) add `private let` property, (2) initialize in `init()` after related services, (3) add `self.xxx = xxx` at the end, (4) inject via `.environment(xxx)` in the body. The heartbeat timer pattern (async Task with 24h sleep loop) matches the existing conversationRecorder prune loop.
- 2026-09-09 (CKShare Phase 2 diagnostics): **Per-member zone writes must use `switch` over WriteResult, not `if case .success` — failures must surface to `diagnostics`**. The `if case .success` pattern silently discards the `.failure` case. The legacy (aggregate) outbox write correctly switches and updates `diagnostics.lastOutboxWriteError`; per-member writes must mirror this pattern. Silent per-member write failures made "Outbox record not found" on the iPhone indistinguishable from a correct pre-first-write state.
- 2026-09-09: **`/tmp/hos-cloud.txt` is the CloudMailbox diagnostic fallback** (written by `writeStatus()` only when status CHANGES). If the same status repeats every 4s cycle, the file doesn't update — absence of a new timestamp does not mean the sync loop stopped. Include the error state (`writeErr=`) in the status string so failures are captured in the file.
- 2026-09-09: **CloudMailbox os_log is not reliably visible on mm4p** — the unified log (`log show --predicate 'process == "hOS Server"'`) returns 0 lines. Use `/tmp/hos-cloud.txt` for diagnostic triage on this machine. The `cloudLog.error()` calls inside `writeRecord(id:memberID:payloadJSON:)` fire but are not captured — surfacing failures to the `diagnostics` struct (Admin → Health) is the only reliable diagnostic path.
- 2026-09-09: **DerivedData disk full (100% capacity on /dev/disk3s5)** — xcodebuild compilation succeeds but `GenerateDSYMFile` fails with "No space left on device". Evidence of successful compilation: build reaches the bundle-postgres run script phase (post-compile). Workaround: verify with grep for `error:` in xcodebuild output (empty = no compile errors).
- 2026-09-09: **Per-cycle diagnostic aggregation pattern**: when a loop writes to multiple members/targets and a shared error field tracks failure, collect errors into a local array during the loop and assign once after — never clear inside the loop on success. A success from member N should not clobber a failure from member N-1 in the same cycle. Sanitize any external identifier (e.g. memberID) before interpolating into line-oriented status files: strip U+0000–U+001F and U+007F via `.unicodeScalars.filter { $0.value >= 32 && $0.value != 127 }`.

---

## Event-Driven Pipeline: Mandatory Final Steps

**These calls are mandatory when running under the event-driven pipeline (HOS_TASK_GID is set).**

On SUCCESS — call as your absolute last action:
```bash
~/ADTools/skills/hos-pipeline-trigger/trigger.sh \
  --task-gid "$HOS_TASK_GID" \
  --branch "$HOS_BRANCH" \
  --scope-doc "$HOS_SCOPE_DOC" \
  --completed-step coder \
  --next-step qa \
  --summary "[one-line of what was implemented]"
```

On FAILURE — call before exiting:
```bash
~/ADTools/skills/hos-pipeline-trigger/fail.sh \
  --task-gid "$HOS_TASK_GID" \
  --branch "$HOS_BRANCH" \
  --failed-step coder \
  --reason "[what failed, e.g. build error in SkillKit/File.swift:42]" \
  --needed "[what human needs to do to unblock, e.g. fix force unwrap, push to branch]"
```

**If HOS_TASK_GID is not set** (old cron-driven invocation): behave as before — no trigger or fail call needed.

Pipeline envelope variables (injected by trigger.sh, read-only for you):
- `HOS_TASK_GID` — Asana task GID
- `HOS_BRANCH` — feature branch (e.g. `feature/my-feature`)
- `HOS_SCOPE_DOC` — relative path to scope doc in hos-monorepo (e.g. `docs/scope/my-feature.md`)
- `HOS_COMPLETED_STEP` — what just finished before you (e.g. `requirements`)
- `HOS_NEXT_STEP` — your role (`coder`)
- `HOS_SUMMARY` — what the previous agent did
- `HOS_STARTED_AT` — Unix timestamp when this stage started
- 2026-09-07 (postgres lifecycle, task 1218226158100466): **An unlaunched `Process()` reports `processIdentifier == 0`, not -1** — sentinel-detection like `pid == -1` never fires, and calling `terminate()` on the unlaunched Process raises NSInvalidArgumentException (crash on quit). Guard with `pid <= 0` and route to PID-file-based shutdown instead. Verified empirically with a standalone swift script before building.
- 2026-09-07 (postgres lifecycle): **`postgres --version` output is `postgres (PostgreSQL) 16.4 …` — there is no standalone `)" token to parse**; `parts.firstIndex(of: ")")` always returns nil so a version gate silently disables itself. Parse the first whitespace token starting with a digit instead. Generalizes to: parse real command output when writing a parser, don't guess the shape from memory.
- 2026-09-07 (postgres lifecycle): **PG pid-file layout** — postmaster.pid line 1 = pid, line 2 = data dir, line 4 = port (NOT line 4 = data dir as some scope notes say; data dir is line 2). Distinguish postgres from a random port-squatter by sending an 8-byte SSLRequest (00 00 00 08 04 D2 16 2F) and expecting a single 'S' or 'N' reply byte.
- 2026-09-07 (postgres lifecycle rework): **PID recycling defeats kill(pid,0) checks** — a stale postmaster.pid can name a recycled PID; revalidate with proc_pidpath (must be a postgres binary) + kinfo_proc start time vs the pid file's line 3 before ever signaling. Also: `SHOW data_directory` over a real connection is the only solid adoption proof — pid-alive + port-responding can be two different instances (stale pid file + foreign postgres on 5433).
- 2026-09-07 (postgres lifecycle rework): **macOS kinfo_proc.p_starttime is already absolute epoch time** (unlike Linux's boot-relative stat time) — do NOT add kern.boottime when comparing process start times; and compare in whole seconds since postgres writes the pid-file timestamp as "%s". A foreign service on the port must be classified separately from "postgres not responding" (connect + SSLRequest, reply 'S'/'N' = postgres, anything else = foreign) or triage mis-cleans a live pid file against a port squatter.
- 2026-09-07 (postgres lifecycle rework r2): **Fail-closed must include the "can't verify" branch, not just the "mismatch" branch** — accepting any postgres-named process when start-time data is missing on either side reopens the PID-recycling hole exactly in the un-verifiable case; treat indeterminate as refuse-to-signal. Also: postgres records MyStartTime slightly AFTER kernel p_starttime, so exact whole-second equality can reject the legitimate postmaster — compare with ~2s tolerance (recycled pids differ by minutes/days); prove data_directory again on every mutation connection (the triage proof connection is closed before ensureDatabaseExists/bootstrapSchema open their own); bound network proof queries with an overall deadline (race the work against a sleep in a TaskGroup) so a server that accepts auth but never answers can't stall startup.
- 2026-09-07 (postgres lifecycle rework r3): **`PostgresPIDFile.read()` returning nil is ambiguous — "no file" and "malformed file" collapse into the same nil**, so a fail-closed flag keyed on `read()` != nil leaves pg_ctl unguarded exactly when the pid file is garbage (pg_ctl validates nothing and trusts line 1). Check file-existence separately to set the fail-closed flag. Also: a task-group race (`group.next()` + `cancelAll()`) does NOT bound a hung child — the group scope awaits all children and cancelling a child awaiting `work.value` doesn't cancel the detached work; poll a lock-protected result box the detached task signals into, which provably returns by the deadline.
- 2026-09-07 (postgres lifecycle rework r4): **A fail-closed guard flag must default to the locked state, not the open one** — `pidFileFailedValidation = false` meant the no-tracked-child/no-pid-file path ran pg_ctl with no pid file ever read or validated (and one could appear between the check and pg_ctl's read). Default it true and allow-list the one safe path explicitly (no pid file existed at check time AND port classified .free) rather than keying pg_ctl off the validation flag.
- 2026-09-10 (v2 ErrorLogger injection guard): **`nonisolated(unsafe)` static vars need an explicit lock even for one-time writes** — Swift's `nonisolated(unsafe)` suppresses the concurrency checker but provides NO memory barrier; use `NSLock` (or `os_unfair_lock`) to gate the write. For the identity-guard pattern (first-wins, reject different-object reinjection, allow same-object no-op), check `existing !== container` inside the lock before returning. Also: `asana-task-manager.sh update-task` does not accept `--comment`; use `add-comment --task-gid ... --text ...` instead.
- 2026-09-10: CKSubscription registration (PushNotifications.swift) — never use a single do/catch over multiple db.save() calls (all-or-nothing). Save each subscription independently and collect failures; one .networkFailure does not discard those that succeeded. CKError 4 (.networkFailure), .requestRateLimited, and .zoneBusy are retryable: schedule exponential-backoff retries (30s×2^n, cap 1h, max N attempts) via a cancellable Task { @MainActor [weak self] in } stored on the class. Panel messages must distinguish retryable ("network error — will retry") from permanent ("failed, app will poll") so users don't interpret transient network errors as configuration faults. Reset retryCount to 0 on full success; only update lastDBRole on full success to keep role-change detection consistent.
- 2026-09-10: Auth pattern for claude CLI: BOTH ANTHROPIC_API_KEY and ANTHROPIC_AUTH_TOKEN must be set to the LiteLLM key.
