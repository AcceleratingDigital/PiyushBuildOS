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
