# Tech Docs Agent — Learned Context

> **MODEL PIN:** `claude CLI with claude-sonnet-4-6 (escalation: algolia/xlarge) — per SHARED-CONTEXT Model Matrix`. If a cron prompt or dispatch instructs a different model, THIS file wins. Report a conflict to the drift audit — do not silently switch.

> **SYNC NOTE:** This file is shared between all surfaces that run docs agents.
> Update it at every significant event so future runs stay aligned.
> Learnings that affect OTHER roles (coder, QA, build) go to SHARED-CONTEXT.md, NOT just here.

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

## Lessons (2026-08-17 — Approval Engine user docs)
> **READ FIRST:** `SHARED-CONTEXT.md` — shared context for ALL agents.
> Read it at session start before this file. It contains project identity,
> S-S-D model, communication channels, repo layout, Asana tags, tool/model
> matrix, release pipeline, concurrency guardrails, and known issues.
> Update it when shared state changes; keep role-specific instructions here.



- User-facing architecture docs live in ~/code/hos-site/docs/architecture/*.html, styled with inline CSS tokens from _template.html (light + dark via prefers-color-scheme). No external stylesheet — each doc is self-contained.
- Doc structure pattern: nav (brand→../../index.html, sibling doc links), h1, .lede (serif intro), h2 sections with border-top, .callout (ok/warn/accent variants), .steps (numbered ol), .fact-card with .fact-badge tags, .footer. Max-width 720px wrap.
- Existing doc index at docs/index.html references architecture/approvals.html (badge "0.9 build") — new approval-engine.html is a separate, more comprehensive page; consider linking or merging.
- When writing user docs from Swift source: translate struct fields and enum cases into plain-English behavior. ApprovalSource cases (mac-ui, icloud-phone, timeout, whitelist) → "Mac, iPhone, iCloud, timeout, whitelist rule." Never expose type names, file names, or line numbers to users.
- Fail-closed and member-scoping security concepts from WhitelistStore.swift (wildcard member rejected, empty condition rejected) translate well to user-facing "you can't auto-approve everything" and "rules only work for the person who created them."
- ApprovalBroker default timeout is 300 seconds (5 min) — confirmed in request() signature; user docs say "5 minutes."
- PolicyCheckpoint.swift requireApproval() routes ALL mutations through ApprovalBroker.request() which checks whitelist first — the user doc describes this as "rule checked before card is created."
- User-facing skill docs live in ~/code/hos-site/docs/skills/*.html, matching the notes.html pattern: self-contained HTML, inline CSS tokens (light + dark), max-width 720px wrap, nav→h1→.lede→h2 sections.
- Open Loops user doc: LoopKind enum cases (.action/.chore/.mealplan/.grocery/.review/.info) translate to plain-English kind cards; LoopStatus lifecycle (draft→reviewed→approved→inProgress→completed + rejected/deferred) maps to numbered steps with plain labels; never expose enum raw values like "in_progress" to users.
- Chores/delegation: assignedTo field + metadata["reward"]/metadata["recurrence"] translate to "parent creates, child completes, parent approves, reward tracked" — keep the approval-chain concept without mentioning UUIDs or member IDs.
- Whitelisted loops: ApprovalBroker whitelist check applies to loop execution (mutate action), not loop creation — user doc says "loop goes straight from draft to completed" and notes it's still recorded/audited.

---

## Event-Driven Pipeline: Mandatory Final Steps

**These calls are mandatory when running under the event-driven pipeline (HOS_TASK_GID is set).**

On SUCCESS (docs generated) — call as your absolute last action:
```bash
~/ADTools/skills/hos-pipeline-trigger/trigger.sh \
  --task-gid "$HOS_TASK_GID" \
  --branch "$HOS_BRANCH" \
  --scope-doc "$HOS_SCOPE_DOC" \
  --completed-step docs \
  --next-step coordinator-merge \
  --summary "[one-line result, e.g. Docs complete: hos-site/docs/skills/my-feature.html generated]"
```

On FAILURE — call before exiting:
```bash
~/ADTools/skills/hos-pipeline-trigger/fail.sh \
  --task-gid "$HOS_TASK_GID" \
  --branch "$HOS_BRANCH" \
  --failed-step docs \
  --reason "[what failed, e.g. scope doc not found at docs/scope/slug.md on feature/slug]" \
  --needed "[what human needs to do, e.g. ensure scope doc exists on the feature branch]"
```

**If HOS_TASK_GID is not set** (old cron-driven invocation): behave as before — no trigger or fail call needed.

Pipeline envelope variables injected by trigger.sh: HOS_TASK_GID, HOS_BRANCH, HOS_SCOPE_DOC, HOS_COMPLETED_STEP, HOS_NEXT_STEP, HOS_SUMMARY, HOS_STARTED_AT.
