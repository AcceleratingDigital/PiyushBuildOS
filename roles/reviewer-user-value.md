# User Value Reviewer — Learned Context

Read this file before starting any user value review. It accumulates UX
patterns, past feedback, and accessibility requirements.

## User value checklist

- [ ] Error messages: are they user-friendly? (not "Error: nil" or stack traces)
- [ ] Success messages: do they tell the user what happened clearly?
- [ ] Edge cases: what does the user see when there are zero results?
- [ ] Input validation: does the skill tell the user what's wrong with their input?
- [ ] Accessibility: can the output be read by screen readers? (plain text, no tables)
- [ ] Feature completeness: does the skill do everything the spec says?
- [ ] Consistency: does it match the behavior of similar skills? (e.g., all read skills
      return results in the same format)
- [ ] Helpfulness: if the skill can't help, does it suggest alternatives?

## UX patterns for hOS

- **No results:** Return "No <X> found for '<query>'" — not empty string, not error
- **Missing dependency:** Return "<X> not available. <hint how to enable>" — not crash
- **Permission denied:** Return "hOS doesn't have permission to access <X>. Grant in System Settings > Privacy & Security."
- **Large result sets:** Show count + first N results, not all results
- **Read skills:** Return format: `title — date — "snippet"` (consistent across skills)

## Past feedback

| Date | Skill | Issue | Resolution |
|---|---|---|---|
| 2026-08-15 | NotesRead | Return format clear and consistent | approved |
| 2026-08-15 | JournalRead | "journal n/a" message clear | approved |

## Accessibility

- All skill output is plain text (no HTML, no markdown tables)
- Output is read aloud by VoiceOver compatible with hOS TTS
- Error messages are descriptive, not just error codes

## Lessons from Postgres+pgvector Tier 0 storage refactor review (2026-08-17)

- Embedding dimension mismatch: schema declares vector(768) but fallback NLEmbedding produces 512-dim — INSERT will fail whenever Ollama is absent. Schema must use vector(768) OR pad/truncate embeddings, OR use a separate column per source.
- Graceful degradation works for binary-missing case (.disabled state) but startup failure (initdb/schema error) falls through to .stopped with only a log — stores then throw opaque connection errors, not a user-friendly "database setup failed" message.
- No SQLite→Postgres data migration exists: old memory.db and finance.db data is silently orphaned on upgrade. Must either migrate or show a one-time "your previous data is in ~/Library/Application Support/hOS/*.db" warning.
- backup() function exists but is never called or scheduled — backups don't happen. Even if scheduled, user has no UI visibility that backups exist or how to restore.
- PostgresManager.stop() is never called on app shutdown — postgres child process is orphaned on quit.
- SQL injection risk in unsafeSQL queries: escape() only handles single quotes, not backslashes (relevant if standard_conforming_strings is off). Prefer parameterized interpolation (conn.query("""...""")) over PostgresQuery(unsafeSQL:) wherever possible.
- Dedup-on-write sets op='superseded'/'near-duplicate' but currentCTE filters op <> 'forget' — superseded rows still appear as "current" in recall. The old code filtered op = 'store' only.
- FinanceError.cannotOpen error message says "run PostgresManager.shared.start() first" — this is a developer instruction, not a user-actionable message.
- Lifecycle edge cases handled: approve() allows draft/reviewed/approved (idempotent). reject() and deferLoop() also allow their own current status (idempotent). isValidTransition allows idempotent self-transitions.
- Error messages are clear: invalidStatusTransition returns "Invalid status transition: draft → completed" with HTTP 409. loopNotFound returns HTTP 404. All errors surfaced via sendEnvelope with errorDescription.
- LoopKind includes chore, mealplan, grocery, review, info — spec items C3 (chores), C5 (meal planning), F3 (delegation via assignedTo field) are in the data model even if not fully built.
- Missing: no companion API endpoint for DELETE /loops (store.delete exists but is not wired). Users cannot delete loops via the iOS app — only reject (terminal). Consider adding delete for completed/rejected loops.
=======
## Admin Audit Timeline review (2026-08-17)

- Computed property `filteredRecords` running `allRecords.filter{}` on every view update causes lag with large audit logs — add debounce or paginated fetch. (MED)
- DatePicker with `Date.distantPast` as nil-sentinel is confusing — user can't visually distinguish "no filter" from a real date. Use explicit enable/disable toggle. (LOW)
- Icon-only buttons (copy-to-clipboard) without `accessibilityLabel` are invisible to screen readers — always label interactive elements. (MED)
- Spec compliance requires checking each spec requirement line-by-line — "member (if applicable)" in timeline rows was missed. (LOW)
- Empty state differentiation ("No access yet" vs "No matching entries") is a good pattern — reuse across features. (GOOD)
## Privacy Indicator (U8) review (2026-08-19)

- Badge states that are defined but never produced by any code path are a user-value defect — if yellow/red/hybrid never fires, the badge is decorative and the privacy "guarantee" is illusory. Always trace the producer side, not just the UI.
- When a failover path changes provider mid-request (cloud→on-device), the final `lastProvider` overwrites the original — badge shows green for a request that started in cloud. Track provenance, not just final state.
- "LLM", "external calls", "LiteLLM proxy" are developer vocabulary, not household vocabulary. Info sheets for consumer features need plain-language rewrite ("How your request was handled", "Internet lookups", "Where your data lives").
- SF Symbol names read aloud by VoiceOver ("lock.shield.fill") are incoherent — always set accessibilityLabel on icon+text badge buttons with a human sentence.
- Badge tap target with 7×3pt padding is below Apple's 44pt minimum — use .contentShape(Rectangle()) + minimum touch size on badge buttons.
- iOS badge that is always-green with no per-response context is acceptable IF the architecture guarantees local processing (companion → household Mac server), but the info sheet should explain WHY it's always green, not just state it.

## B7 Plain-Language Approvals review (2026-08-19)
> **READ FIRST:** `SHARED-CONTEXT.md` — shared context for ALL agents.
> Read it at session start before this file. It contains project identity,
> S-S-D model, communication channels, repo layout, Asana tags, tool/model
> matrix, release pipeline, concurrency guardrails, and known issues.
> Update it when shared state changes; keep role-specific instructions here.



- A renderer that accepts `params: [String: String]` but whose primary call site (CloudMailbox) passes `params: [:]` because PendingApproval doesn't store params is a CRITICAL user-value defect — every template with {recipient}/{title}/{date} produces broken strings with stripped placeholders. Always trace the full data flow from producer (PolicyCheckpoint) → carrier (PendingApproval struct) → consumer (renderer), not just the renderer's internal logic.
- Parameter key mismatches between producer and template (e.g., PolicyCheckpoint passes `params: ["name": name]` but template uses `{title}`) silently produce empty interpolation even when params ARE passed. Canonical key names must be documented and enforced on both sides.
- "Loop" is hOS-internal jargon (GTD "open loops"). Non-technical family members need "task", "chore", or "reminder" — never "loop". Always audit template nouns against consumer vocabulary, not developer vocabulary.
- Regex stripping of unreplaced `{token}` placeholders is graceful but produces awkward output ("send an email to " with trailing space, "create a calendar event  on " with double space). Prefer conditional phrasing or fallback text over blind stripping.
- The fallback "The agent wants to {mutation} in {scope}" leaks raw developer terms (e.g., "mutate in finance") to non-technical users. Fallbacks must use human nouns/verbs, not raw enum values.
- "safe to auto-approve" as risk language nudges users toward unsafe automation — prefer neutral language like "low risk" or "routine action" that doesn't implicitly recommend a decision.
- Templates added speculatively (loop:create/update/delete) with no corresponding requireApproval call site are untestable in the real flow and may rot. Always verify a template has a live producer before shipping.
- Rule preview broad-rule warning only fires when selectedFields is completely empty — non-empty fields with unrecognized keys fall through to a conditionless preview with no warning. Broad-rule detection should check whether a recognizable condition was produced, not just whether input was empty.
