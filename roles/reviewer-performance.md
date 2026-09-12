# Performance Reviewer — Learned Context

> **MODEL PIN:** `codex CLI with algolia/xlarge (escalation: claude-sonnet-4-6) — per SHARED-CONTEXT Model Matrix`. If a cron prompt or dispatch instructs a different model, THIS file wins. Report a conflict to the drift audit — do not silently switch.

> **SYNC NOTE:** This file is shared between all surfaces that run performance reviews.
> Update it at every significant event so future runs stay aligned.
> Learnings that affect OTHER roles (coder, QA, build) go to SHARED-CONTEXT.md, NOT just here.

Read this file before starting any performance review. It accumulates perf
patterns, past issues found, and hOS resource constraints.

## Performance checklist

- [ ] Algorithmic complexity: any O(n²) or worse loops on large datasets?
- [ ] Memory: does the skill load entire datasets into memory?
- [ ] I/O: are there unnecessary file reads or network calls?
- [ ] Concurrency: are async operations truly parallel or accidentally serial?
- [ ] Scaling: what happens with 10,000 entries? 100,000?
- [ ] Timeout: can the skill hang indefinitely on a slow operation?
- [ ] Resource leaks: are file handles, processes, or connections properly closed?

## hOS resource constraints

- **48GB RAM** on dev machine, but user machines may have 8-16GB
- **Apple Silicon** — efficient but skills should not peg CPU for > 5 seconds
- **No background processing** — skills run on-demand, must return quickly
- **File-based data** (Obsidian, notes) can be large — don't load all files into memory

## Known performance facts (from real testing — check against these)

These are proven facts. If a skill violates one, flag it:

1. **JXA is slow.** osascript process spawn + bridge overhead is significant.
   If a skill uses JXA where a SQLite or native framework path exists → flag as MED/HIGH.
2. **Email should use SQLite** (`~/Library/Mail/V*/MailData/Envelope Index`), NOT JXA.
3. **Calendar/Reminders should use EventKit** (Swift), NOT JXA.
4. **Contacts should use Contacts framework** (Swift), NOT JXA.
5. **Messages should use SQLite** (`~/Library/Messages/chat.db`), NOT JXA.
6. **Notes: JXA is the only option** — no framework, private SQLite. Accept it but
   bound the scan (max 100 notes).
7. **Spotlight should use mdfind** (indexed), not manual FileManager enumeration.
8. **Obsidian: FileManager direct** — no JXA needed.

See `docs/agent-context/coder.md` § "Quick reference: data source → fastest access method" for the full table.

## Patterns to check

1. **Full-content loading** — reading entire file contents for search is O(n) memory.
   For large vaults, consider streaming or indexing. Found in: JournalRead v1 (LOW).
2. **Lowercasing entire strings** — `string.lowercased()` creates a copy. For large
   texts, use `string.lowercased().contains()` sparingly or use case-insensitive comparison.
3. **Process spawning** — each `Process()` call has overhead. Batch JXA calls if possible.
4. **File enumeration** — `FileManager.enumerator` is lazy, but collecting all URLs first
   then iterating is memory-heavy. Iterate lazily.

## Past findings

| Date | Skill | Severity | Issue |
|---|---|---|---|
| 2026-08-15 | JournalRead | LOW | Full-content loading + lowercasing of every .md file |
| 2026-08-15 | JournalRead | LOW | No early termination beyond limit check |
| 2026-08-15 | NotesRead v3 | HIGH | Unbounded scan — iterates all notes, no max-scan ceiling. limit only slices output, not iteration |
| 2026-08-15 | NotesRead v3 | MED | Full plaintext loaded for every note, not just matches |
| 2026-08-15 | NotesRead v3 | MED | No process timeout — can hang indefinitely |

## When to block vs iterate

- **BLOCK:** 10x+ slower than reasonable, will crash on large inputs, hangs indefinitely
- **ITERATE:** Could be faster, but current performance is acceptable for typical use

## Postgres + pgvector refactor (2026-08-17)

- Actor-isolated persistent PostgresConnection is safe but serializes ALL operations — any blocking HTTP call (Ollama embeddings) inside the actor locks the entire store. Run embedding computation OUTSIDE the actor or pre-compute and pass in.
- Dedup-on-write (nearest-neighbor SELECT + conditional UPDATE + INSERT) = 3-4 sequential queries per write. Consolidate into a single SQL DO block / CTE to cut round-trips.
- Hybrid recall (keyed ILIKE → embedding HTTP → pgvector cosine → keyword ILIKE) = 3 DB queries + 1 HTTP call, all sequential. The CTE (ROW_NUMBER window) is recomputed per query — compute once.
- HNSW index is used correctly: `ORDER BY embedding <=> $query_vector ASC LIMIT n` triggers index scan. Partial index `WHERE embedding IS NOT NULL` is good.
- FinanceStore.commit() inserts transactions one-by-one in a loop — same as SQLite era, not a regression, but Postgres multi-row INSERT or COPY would be 10x faster for large imports.
- assertScopesUnchanged does N+1 queries (one SELECT per account) — should be a single SELECT ... WHERE fingerprint = ANY(...).
- Health check (pingSucceeds) correctly uses a SEPARATE connection (id: 0) that it opens/closes per check — does not steal the store's persistent connection.
- PostgresManager.start() is non-blocking on app launch (wrapped in Task). Stores lazy-connect on first use — may get errors if accessed before Postgres is ready.
- PostgresSchema.bootstrap() defer close is fire-and-forget (Task { conn.close() }) — connection lingers briefly. Acceptable for one-time bootstrap.
- Missing index: accounts.member_scope has no index — fine for <100 accounts, needs one at scale.

## Approval Engine F2 whitelist (2026-08-17)

- WhitelistStore.checkWhitelist() issues a fresh Postgres SELECT on EVERY mutation — no in-memory cache. Rules are small/rarely-changing; cache them in the actor with write-invalidation (createRule/deleteRule/disableRule bumps a version or replaces the cache).
- checkWhitelist SQL filters only on (member_scope, enabled) — fetches ALL enabled rules for the member then O(n) scans in Swift. Push skill_id/action filtering into the WHERE clause to shrink the result set.
- ApprovalBroker is @MainActor; awaiting WhitelistStore actor for the DB query stalls the MainActor (UI, pending approvals) for each round-trip. Cache would make this instant; alternatively move the check off MainActor.
- WhitelistStore is a single actor with one PostgresConnection (id: 4) — serializes all access. Hot-path checkWhitelist contends with admin allRules/createRule. Acceptable for beta single-owner, bottleneck at scale.
- CloudMailbox polls every 4s and now calls allRules() (Postgres SELECT) on EVERY poll for iCloud sync — 15 queries/min even when rules unchanged. Gate on a dirty-flag/version-counter set by writes, or poll less frequently.
- CompanionServer delete fetches ALL rules then O(n) scan to verify ownership — should be a targeted `SELECT 1 WHERE id=$1 AND member_scope=$2`.
- whitelist_rules table has no TTL/cleanup but growth is bounded (human-created rules, deleteRule is permanent). Acceptable.
- Index idx_whitelist_member (member_scope, enabled) covers the query but not skill_id/action — covering index would enable index-only scans at scale.
- Manual SQL escaping via replace(' → '') instead of parameterized queries — negligible perf impact but parameterized is safer and slightly faster.
- OpenLoopStore.children(of:) query has no LIMIT clause (line 219) — unbounded result set if a parent loop has thousands of children (e.g., grocery items). Add LIMIT + pagination.
- OpenLoopStore.update() runs N separate UPDATE queries (one per field, lines 295-332) instead of one combined UPDATE — 6 round-trips worst case. Acceptable for 1-6 fields but note for v2.
- OpenLoopService is @MainActor but delegates all DB work to the OpenLoopStore actor (good pattern — no main-actor blocking).
- Loop mutation broadcasts on WebSocket (broadcastWebSocket) per mutation — bounded by maxConcurrentWebSockets=32 and pruned of dead sessions. Not a storm risk.

## Shared Capabilities Architecture review (2026-08-17, commit 4c433fa)

- **ClassificationRuleStore.match() loads ALL domain rules from Postgres on EVERY classify() call** — no in-memory cache. Same anti-pattern as the flagged WhitelistStore.checkWhitelist(). Rules are small/rarely-changing; cache in the actor with write-invalidation (create/update/delete bumps a version). (MED — v2)
- **classify() runs caller-supplied rules then DB rules then LLM** — three-stage, each potentially scanning O(n) rules with regex compilation per call. NSRegularExpression compilation is not cached (recompiled every match). Pre-compile and cache regexes per rule. (MED — v2)
- **emitKnowledge emits entities one-at-a-time** — each entity is a separate `memory.remember()` call (embedding HTTP + INSERT). No batching of embeddings or multi-row INSERT. N+1 round-trip risk for large entity batches. Acceptable for small batches, note for v2. (LOW — v2 batch embeddings + multi-row INSERT)
- **summarize/extract/classify LLM calls have no explicit timeout** — `quickComplete` relies on the underlying URLSession timeout (Ollama 120s). A slow/stuck Ollama blocks the @MainActor skill loop for up to 120s per call. No `withTimeout` wrapper at the shared-capability layer. (MED — v2: wrap LLM calls in a withTimeout)
- **Content truncation limits are reasonable** — 8000 chars (summarize/extract), 4000 chars (classify). Good context-budget hygiene.

## Semantic Memory Layer — Mail SQLite→Postgres migration (2026-08-17, commit 6615b97)
> **READ FIRST:** `SHARED-CONTEXT.md` — shared context for ALL agents.
> Read it at session start before this file. It contains project identity,
> S-S-D model, communication channels, repo layout, Asana tags, tool/model
> matrix, release pipeline, concurrency guardrails, and known issues.
> Update it when shared state changes; keep role-specific instructions here.



- MailClassificationStore uses `PostgresQuery(unsafeSQL:)` with manual `esc()` (single-quote doubling) instead of parameterized binds — same anti-pattern noted in WhitelistStore. Negligible perf impact but unsafe and slightly slower (PostgresNIO can't cache prepared plans).
- Single lazy PostgresConnection (id: 3) in the actor serializes ALL mail store ops — same pattern as MemoryStore (id: 2). Fine for beta single-owner; contention risk at scale. No connection pool.
- MailSkill.runTriageCycle (limit ≤100) calls `isClassified` then `storeClassification` per header in a loop = up to 200 sequential actor round-trips per triage. N+1. Should batch-check already-classified message_ids with `SELECT ... WHERE message_id = ANY(...)` and multi-row INSERT.
- MailLearningLoop.runCycle calls `storeObservation` per sender pattern in a loop — N sequential upserts (N = distinct patterns). Batch with multi-row INSERT ... ON CONFLICT.
- SystemSQLiteReader.withReadonlyDatabase copies the entire external DB to temp on EVERY call. Envelope Index can be 100s of MB. Synchronous `FileManager.copyItem` blocks the MailProvider actor. No caching/mtime check — re-copies even if unchanged within a single runCycle that calls unreadCount + recentHeaders + accountStats (3 copies). HIGH.
- accountStats tries 3 SQL variants sequentially, breaking on first non-empty result — if the first query returns 0 rows (valid for empty mailbox), it falls through to the next. Wasteful but bounded.
- PostgresNIO `rows.decode(...).for try await` streams rows (AsyncSequence), not bulk-loaded — good. Results accumulated into arrays bounded by LIMIT. Fine.
- PostgresSchema mail indexes: idx_mc_label(label, date_received) + idx_mc_classified(classified_at). isClassified uses message_id (PK, covered). No index on `sender` — but no query filters on sender alone currently. Will need one if sender-based queries are added.
- No Task.sleep/busy-waits/unbounded concurrency in the new code. Good.
- Blocking file I/O (FileManager.copyItem, sqlite3_open, sqlite3_step) runs synchronously inside the MailProvider actor — off MainActor (good) but blocks the actor's cooperative thread pool for the full copy+query duration. For large Envelope Index this can be seconds.

## PostgreSQL not-starting fix (2026-08-22, feature/postgres-not-starting)

### waitForRunning polling pattern
- `waitForRunning(timeout:)` polls every 0.5s via `Task.sleep` on the PostgresManager actor. Since PostgresManager is an actor, each poll hop is an actor-isolated await — but the caller (e.g. a `Task` in hOS_ServerApp) is NOT on the main thread, so no main-thread blocking. Correct.
- The method CANNOT use a continuation/AsyncStream easily because `state` is set by `start()` on the same actor — there's no external mutation to observe. A `CheckedContinuation` resumption would need to be wired into `start()`'s success/failure paths. The 0.5s poll is acceptable for a one-time startup gate (max 60 iterations). Flag as LOW (could be event-driven for cleaner design, but perf impact is negligible — it's a one-shot wait).
- **Initial state race**: `state` defaults to `.disabled` (line 38). If `waitForRunning` is called BEFORE `start()` begins executing on the actor, `isDisabled` returns true immediately → returns false instantly. In practice `start()` is enqueued first, but actor scheduling is not guaranteed ordered. This is a correctness race, not a perf issue, but worth noting.

### Startup sequencing — replaced fixed sleeps
- OLD: `Task.sleep(15s)` before pruneOldEpisodes, `Task.sleep(10s)` before geofenceMonitor. These blocked for the FULL duration even if Postgres started in 2s.
- NEW: `waitForRunning(timeout: 30)` — returns as soon as `.running` is observed. Strictly better for the common case (fast startup). Worst case is 30s (same as before, roughly).
- No deadlock risk: `waitForRunning` returns false on `.disabled`/`.stopped` — it won't wait the full 30s if Postgres fails fast. The conversation prune and geofence monitor proceed regardless (return value is discarded with `_ =`), which is correct — they should run even if DB is down.
- **Race condition**: `start()` is fire-and-forget (`Task { await PostgresManager.shared.start() }` at line 159). The two `waitForRunning` calls at lines 301/393 may begin before `start()` has had a chance to flip state from `.disabled` to `.initializing`. If `start()` hasn't started executing yet, `isDisabled` is true (initial state) → `waitForRunning` returns false immediately → prune/geofence proceed without DB. This is the same initial-state race noted above. LOW severity for prune (runs again on schedule), MED for geofence (won't retry until app restart).

### Bundle script — build time impact
- `alwaysOutOfDate = 1` in pbxproj means the script runs on EVERY build, including incremental. No input/output paths declared, so Xcode can't skip it.
- pgvector is cached in `/tmp/pgvector-build` — the `git clone` + `make` only runs once (if `/tmp` persists). Subsequent builds skip to the copy step. Good.
- However, `dylibbundler -od` (overwrite-directory) runs on EVERY build for all 4 binaries. dylibbundler scans Mach-O dependencies, copies dylibs, rewrites load commands — this adds ~2-5s per build. Acceptable but not zero.
- The binary copies (`cp -f`) and `codesign --force --sign -` also run every build — another ~1-2s. Total overhead ~3-7s per build. LOW.
- **MED concern**: `/tmp/pgvector-build` is cleared on reboot. First build after reboot does a full `git clone` + `make` — ~30-60s with network. Not cached in DerivedData. Should use a persistent location (e.g. `DERIVED_DATA/...` or `SRCROOT/.build-cache/`).

### DB connection races — stores don't call waitForRunning
- NONE of the 19 stores that check `isDisabled` call `waitForRunning` before connecting. They only check `isDisabled` and immediately attempt `PostgresConnection.connect`. If Postgres is still in `.initializing`/`.starting` state (not disabled, not running), the store will attempt a connection that fails because the server isn't accepting connections yet.
- `waitForRunning` is only called in 2 places: conversation prune and geofence monitor — both in hOS_ServerApp startup Tasks. Stores that are accessed later (e.g. FinanceStore when user opens finance) are fine because Postgres will have started by then. But stores accessed during early startup (MemoryStore for daily brief, AuditStore for skill logging) can still race.
- This is the SAME race the scope doc identified in root cause #5, and the fix only partially addresses it. The readiness gate exists but isn't wired into the store connection path. MED.

### Admin Health tab polling
- `HealthSection.refreshDbStatus()` is called once via `.task` modifier — it does NOT poll. It fetches status once when the view appears. No timer, no repeat. Good — no performance concern.
- However, the one-shot fetch means the status can be stale: if Postgres starts after the user opens Health, the card still shows "Starting…" until they navigate away and back. LOW (UX, not perf).
- `databaseStatusText` computed property (line 171) is declared but never used — `DatabaseStatusCard` has its own `statusLine` computed property. Dead code. LOW.

## PostgreSQL not-starting fix — QA follow-up review (2026-08-22, feature/postgres-not-starting post-QA)

- `startAttempted` flag correctly fixes the initial-state race: set synchronously at top of `start()` before any await, actor serializes so waitForRunning sees it. Prior LOW/MED finding RESOLVED.
- All 12 stores now call waitForRunning(timeout:30) instead of isDisabled check — prior "stores don't call waitForRunning" MED finding RESOLVED.
- waitForRunning 0.5s poll is acceptable: max 60 iterations, one-shot startup gate, Task.sleep yields cooperative thread — no busy-wait, no CPU spin. LOW.
- /tmp/pgvector-build still cleared on reboot — prior MED finding still valid, not addressed in this branch.
- dylibbundler -od runs every build (alwaysOutOfDate=1) — prior LOW finding still valid; ~3-7s overhead acceptable for correctness guarantee.
- databaseStatusText computed property on PostgresManager (line 182) is dead code — AdminSurface uses its own DatabaseStatusCard.statusLine. LOW.
- AdminSurface HealthSection refreshDbStatus is one-shot (.task modifier), no polling — no perf concern, but status can be stale until view re-appears. LOW (UX).
- waitForRunning discards return value in hOS_ServerApp (lines 301, 393) — correct: prune/geofence proceed regardless of DB state. No regression.
- No unbounded loops, busy-waits, or memory issues in new code. healthCheckTask uses 30s sleep + cancellation check. Crash restart bounded at 3/hr.
