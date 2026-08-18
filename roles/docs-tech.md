# Tech Docs Agent — Learned Context

Read this file before writing any technical documentation. It accumulates
doc structure patterns, SKILL.md format, and code documentation conventions.

## What tech docs to write

- **SKILL.md** — per-skill developer documentation (manifest, capabilities, inputs, outputs, dependencies)
- **QUICK.md** — quick start for each skill (how to invoke, expected output)
- **Architecture docs** — system-level documentation in `docs/`
- **Code comments** — only for non-obvious logic

## SKILL.md format

```markdown
# <Skill Name>

## Manifest
- ID: com.acceleratingdigital.hos.skill.<name>
- Version: <version>
- Capabilities: <list>
- Inputs: <list with types and defaults>

## Description
<what the skill does>

## Dependencies
- <list or "none">

## Security
- <data accessed, permissions needed, TCC requirements>

## Errors
- <list of error conditions and messages>
```

## Conventions

- Write for developers, not users (user docs are the marketing/docs agent's job)
- Include code examples where helpful
- Reference source files by relative path
- Keep under 100 lines per skill doc
- Update docs/08 if new dependency added

## Past work

| Date | Skill | Doc | Notes |
|---|---|---|---|
| 2026-08-15 | NotesRead | SKILL.md | Created during build |
| 2026-08-15 | JournalRead | SKILL.md | Created during build |
