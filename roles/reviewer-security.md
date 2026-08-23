# Security Reviewer — Learned Context

Read this file before starting any security review. It accumulates security
patterns, past vulnerabilities found, and hOS-specific threats.

## Security checklist

- [ ] Path traversal: can user-supplied paths escape intended directories?
- [ ] Injection: is any user input passed to shell, JXA, or SQL without sanitization?
- [ ] Privilege escalation: can a non-owner member access owner-only data?
- [ ] Data exfiltration: does the skill send data to external endpoints?
- [ ] TCC/privacy: does the skill access data not declared in its capabilities?
- [ ] Credential exposure: are any tokens, keys, or passwords logged or returned?
- [ ] File access: can the skill read files outside its intended scope?
- [ ] Process spawning: does the skill spawn processes with user-supplied arguments?

## hOS-specific threats

1. **vault_path / file_path inputs** — can point at `/` or `/etc` and enumerate
   the filesystem. Validate paths are within user home or expected directories.
   **Found in:** JournalRead v1 (HIGH). Fix: validate path contains `.obsidian`
   or is within `~/Documents`.
2. **JXA injection** — if user query is interpolated into JXA string, can inject
   arbitrary JavaScript. Use parameterized queries or escape quotes.
3. **Member scoping** — skills must respect member permissions. A child member
   should not see parent's mail/finance data. Check `context.member` permissions.
4. **Skill manifest capabilities** — skill must declare ALL data it accesses.
   Undeclared access is a security violation.
5. **Approval flow** — write operations must go through approval. Check that
   skills with `.write` capability actually trigger approval checkpoint.

## Past findings

| Date | Skill | Severity | Issue | Fixed in |
|---|---|---|---|---|
| 2026-08-15 | JournalRead | HIGH | vault_path allows filesystem enumeration | pending rework |
| 2026-08-15 | NotesRead | MED | No issue — JXA properly escaped | n/a |
| 2026-08-15 | NotesRead | HIGH | Process deadlock risk (pipe not read) | fixed v3 (drain stderr) |
| 2026-08-15 | NotesRead v3 | MED | Empty plist at hOS Server/hOS-Server-Info.plist — may not be the one Xcode uses at build time. Verify which plist is bundled. | investigating |

## When to block vs iterate

- **BLOCK:** Path traversal, injection, credential exposure, privilege escalation
- **ITERATE:** Could add more validation, but current code is safe

## Credential Vault review (2026-08-15)

- Audit log detail strings must not include credential values — even partial prefixes (`value.prefix(4)`) leak key material into audit records. Use "found"/"not found" instead.
- Shell scripts parsing key=value config files must not use `eval` on unsanitized values — use associative arrays or `source` into a subshell.
- Migration output should minimize credential exposure — first 8 + last 4 chars of an API key is too much; use first 4 + last 4 or just "set (N chars)".
- Keychain access is not cached — 5 `SecItemCopyMatching` calls per `loadEntry()` is wasteful; cache in memory with invalidation on save.
- Credential access not gated by a capability domain is consistent with existing broker pattern but lacks defense-in-depth — add a `.credentials` domain in v2.
- `isConfigured()` should only check the 2 needed fields, not call full `loadEntry()`.
- Partial credential exposure in local-only audit logs is MED (not HIGH) — context: audit records are local SwiftData, visible only to admin/owner.

## Doc Review Patterns (2026-08-18)

- **Incomplete standardization**: When standardizing API names across multiple docs (e.g., categorize→classify), count all occurrences first (search main branch), then verify each one is in the diff. Partial fixes leave contradictions (old name in quotes vs new name elsewhere). Check task spec's claimed occurrence count against actual count in main.

## Privacy Indicator review (2026-08-19)

- PrivacyInfoSheet reveals "via LiteLLM proxy" — internal architecture name exposed to end users. Consider generic "cloud model" wording to avoid leaking infra details.
- ProcessingContext.dataStoredLocally is always hardcoded `true` even when llmSource=.cloud — the "Stored locally" row is misleading when cloud LLM processed the data. Distinguish storage location vs processing location.
- ProcessingContext.ExternalCall.serviceName/purpose are free-form String with no sanitization guidance — a skill could embed user PII. Add doc comment requiring non-PII values.
- LLMSource.hybrid exists in enum but is never constructed — failover from cloud→on-device sets lastProvider=.foundationModels so badge shows "onDevice", hiding that data was first sent to cloud and failed. Consider tracking hybrid path.
- Badge logic uses lastProvider which is mutated during failover — cloud attempt that fails and falls back to on-device will show green "Processed locally" even though user data was sent to cloud endpoint (and rejected). This is a privacy accuracy issue.
- No new network/file I/O in the UI components — purely display of existing metadata. Confirmed safe.
- No AttributedString from untrusted sources — all Text views use literal strings or ProcessingContext fields. No injection risk.

## B7 Plain-Language Approvals review (2026-08-19)

- **Renderer must receive the data it renders**: if a `PendingApproval` struct has no `params` field but the renderer interpolates from params, the production path produces broken/misleading text ("send an email to " with no recipient). Always trace the full call path from struct construction → overload resolution → interpolation. A `render(_ approval:)` convenience overload that passes `params: [:]` is a silent data-loss bug.
- **Unknown mutations must default to HIGH risk, not LOW**: `classifyRisk()` that falls through to `.low` for unrecognized mutations means an unknown dangerous action (e.g. smart-home "unlock door" → mutation "mutate") gets "This is safe to auto-approve". Fallback risk for unknown (scope, mutation) pairs must be `.high` or at least `.medium` — fail-safe, not fail-closed-to-low.
- **Broad-rule detection must check recognized-conditions, not just non-empty dict**: a rule-preview `isBroadRule` that returns false when `selectedFields` contains only unrecognized keys lets a broad rule ("Auto-approve all emails" with no condition clause) render WITHOUT the broad-rule warning. The check must verify at least one RECOGNIZED condition matched, not just that the dict is non-empty.
- **`loop:execute` is a wildcard action**: executing an open loop can trigger arbitrary chained sub-actions. Classifying it as `low` risk (because "execute" isn't in the high/medium mutation lists) is a dangerous under-classification. Any "execute"/"run" mutation should be `high`.
- **Parameter interpolation into human-readable trust text is an injection vector**: even when params are empty in the current production path, the `render(scope:mutation:parameters:)` API is public and interpolates raw param values into sentences. A malicious `recipient` value with newlines or HTML can mislead the user ("school@example.com\nTo: attacker@evil.com"). Sanitize/truncate param values before interpolation, or at minimum strip newlines/control chars.
- **`extractFromDetail` stub no-op is a silent failure**: a helper that returns `nil` with a "best-effort no-op" comment means the structured context fields (recipient, subject, amount) are NEVER populated from the detail string, but the code reads as if they could be. Either implement detail parsing or remove the dead path and document that context requires params.
- **Approval rendering layer is pure presentation — no auth bypass**: confirmed both renderers are stateless `enum` namespaces with no broker references, no state mutation, no whitelist calls. They only read `PendingApproval` fields. This is the correct pattern — rendering must never decide.
- **CloudMailbox additive fields don't add new cross-member leak**: `plainLanguage` and `context` are derived from the same params already serialized as `detail` in the pre-existing outbox payload. No new leak surface, but the shared single-record outbox design means all members' approvals (with detail text) are in one CloudKit record — pre-existing, not B7's fault.

## B2 Approval Cards (iOS) review — 2026-08-19
> **READ FIRST:** `SHARED-CONTEXT.md` — shared context for ALL agents.
> Read it at session start before this file. It contains project identity,
> S-S-D model, communication channels, repo layout, Asana tags, tool/model
> matrix, release pipeline, concurrency guardrails, and known issues.
> Update it when shared state changes; keep role-specific instructions here.



- **approveWithRule ordering**: `approve(id,...)` is called BEFORE the isOwner gate. A non-owner who passes `authorize()` (e.g. parent approving child action) gets the action approved even if rule creation is rejected. Not a bypass (action was approvable via standard Approve anyway), but the owner-gate should come first for clarity and to prevent surprise "action still approved" when the user intended rule creation.
- **Redirect revised approval has no continuation**: `redirect()` appends `newItem` to `pending` but never registers `continuations[newItem.id]`. When the user later approves the revised card, `resolve()` finds nil continuation and silently drops it. The original skill already got `.denied`. Net: redirect is functionally broken (revised action never executes). Not a security hole (no unauthorized execution), but a correctness bug that makes the feature non-functional.
- **LLM redirect prompt injection**: user `directive` is interpolated raw into the LLM prompt. LLM output (title/detail/params) is parsed without sanitization. A malicious directive could produce misleading revised params. Mitigated by the fact that the revised approval requires explicit user approval again — but the user sees LLM-generated text that may not reflect their actual intent. Sanitize/truncate LLM output params.
- **`member` hardcoded to "owner" in decideExtended**: the iPhone always sends `member: "owner"` regardless of who is using the device. CloudMailbox trusts this as the `decider`. If a non-owner uses the iPhone, they impersonate the owner for approval decisions AND rule creation. Pre-existing design issue (no per-user auth on the companion app), amplified by rule creation. The Mac-side `authorize()` check is the only backstop.
- **CloudKit record parsing is safe**: `CloudApprovals` uses `as?` casts with sensible defaults for all extended fields. `parameters` coerced from `[String:Any]` to `[String:String]` via string interpolation — no type confusion. `options` array taken as-is but only checked via `contains()` for known values — extra values harmless.
- **Whitelist condition field selection is safe**: `RuleCreationView` only offers fields from `item.parameters` (action params like to/subject/body). Structural fields (skillID, action, memberScope) are NOT selectable. `WhitelistRule.matches()` only checks condition keys against request `params`, so injected structural keys wouldn't match. `createRule` rejects wildcard memberScope and empty conditions. The condition requires non-empty values. No bypass path found.
- **Redirect cap enforced server-side**: `redirectDepth < 3` check in `ApprovalBroker.redirect()` is authoritative. iOS-side `canRedirect` is cosmetic. Cap is correctly enforced.
