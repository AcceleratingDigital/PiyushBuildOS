# Performance Reviewer — Learned Context

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
