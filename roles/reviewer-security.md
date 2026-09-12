# Security Reviewer — Learned Context

> **MODEL PIN:** `codex CLI with gpt-5.6-sol — per SHARED-CONTEXT Model Matrix`. If a cron prompt or dispatch instructs a different model, THIS file wins. Report a conflict to the drift audit — do not silently switch.

> **SYNC NOTE:** This file is shared between all surfaces that run security reviews.
> Update it at every significant event so future runs stay aligned.
> Learnings that affect OTHER roles (coder, QA, build) go to SHARED-CONTEXT.md, NOT just here.

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

## B8 Audit Timeline (iOS) review — 2026-08-19

- **Owner-fallback in member lookup silently returns wrong member's data**: `members.member(id:)` falls back to `owner` for unknown IDs. `auditScope()` uses `members.member(id: member)?.id ?? member`, so an unknown `member` query param resolves to the owner's ID and returns the owner's audit history instead of an error. Only owner/parent can reach the endpoint, so impact is LOW — but the pattern of "unknown input silently maps to a privileged account" is a recurring anti-pattern. Prefer strict lookup (return nil/400 for unknown member IDs) in audit scoping paths.
- **Member filter not pushed to SwiftData predicate**: `AuditStore.list()` fetches all date-range records then filters by `memberID` in memory (`records.filter`). For one-day `/audit/today` this is fine; for wide `/audit/range` queries it loads every member's records into memory. Push `memberID` into the `#Predicate` to let SwiftData filter at the store layer.
- **No pagination or result cap on `/audit/range`**: unbounded date ranges return the entire matching audit table. Add a max-records limit (e.g. 500) or cursor pagination before this endpoint sees real historical-browsing use.

## B5 Kid Surface review — 2026-08-20

### Findings

- **[MEDIUM] LLMService.swift:1687 — `buildTools()` does NOT filter tools for child role.** `memoryTools + loopTools + choreTools + financeTools + briefTools + host.skills` are all exposed to the LLM regardless of `currentMemberRole`. Although `autoRecall()` is correctly skipped for children (line 1279) and `refreshSystemMessage()` swaps in the kid-safe prompt (line 1230), the LLM can still CHOOSE to call `memory_recall` as a tool during the tool loop. `handleMemoryTool` (line 1842) calls `memory.recall(query:scopes: memoryScopes)` with no child-role guard — `memoryScopes` is hardcoded to `["member:owner", "household"]` (line 153). The kid-safe system prompt says "Never share information about parents..." but this is a soft LLM guardrail, not a hard server-side block. A determined child could prompt-inject the LLM to call `memory_recall` and exfiltrate owner/household memories. FIX: either (a) filter `buildTools()` to exclude `memoryTools` (and `financeTools`, `briefTools`) when `currentMemberRole == .child`, or (b) add a `guard currentMemberRole != .child` in `handleMemoryTool` for `memory_recall`/`memory_remember`/`memory_directive_set` that returns a refusal. Option (a) is stronger — tools not present in the API payload cannot be called at all.

- **[MEDIUM] LLMService.swift:1941 — `handleChoreTool` resolves `isParent` from client-supplied `arguments["member"]`, not from `access.acting.role`/`currentMemberRole`.** The `member` field comes from the LLM tool-call arguments (ultimately from the chat context, which is influenced by the client). If `member` is nil/empty or "owner", `isParent` defaults to `true` (line 1943-1944). A child whose LLM context includes `member: "owner"` or omits the member field can call `create_chore` and `approve_chore`. The companion `/chat` endpoint correctly passes `access.acting.role` to `reply(to:memberRole:)`, but `handleChoreTool` ignores `currentMemberRole` and trusts the LLM-supplied `member` argument instead. FIX: use `currentMemberRole` for `isParent` determination, not the tool argument. The `/loops/complete` HTTP path (line 1066) correctly uses `access.acting.role` — the LLM tool path should match.

- **[MEDIUM] LLMService.swift:2009 — `complete_chore` scope check is bypassable when `member` is nil/empty/"owner".** The assignee check at line 2009 only fires `if let member, !member.isEmpty, member != "owner"`. If the LLM omits the `member` arg or sets it to `"owner"`, the check is skipped entirely and any chore can be marked `kidCompleted`. A child could prompt the LLM to complete another kid's chore. FIX: enforce the assignee check using `currentMemberRole` and the authenticated member ID, not the tool argument.

- **[LOW] CompanionServer.swift:536-541 — child-role block allows `/loops/complete` but not `/loops/approve`, `/loops/reject`, `/loops/defer`, `/loops/edit`.** The block carves out `/loops/complete` (line 538: `!rawPath.hasPrefix("/loops/complete")`). This is correct — kids need to complete chores. But `/loops/approve` and `/loops/reject` are blocked by the prefix `/loops` (since they don't start with `/loops/complete`). This is the desired behavior (only parents approve), but the asymmetry is worth noting: a child CAN reach `handleCompleteLoop` (line 1042), which for the standard (non-chore) loop path calls `openLoops.complete(id:memberScope: scope)` with the child's own scope. This is safe — a child can only complete their own loops. No issue, but the carve-out logic is fragile: any future `/loops/kid-*` route would be blocked unless explicitly carved out. Consider a whitelist approach for kid-allowed loop routes instead.

- **[LOW] CompanionServer.swift:1380 — `handleChoreRewards` has NO role check and accepts arbitrary `member` query param.** `GET /chores/rewards?member=X` is reachable by a child (not in the blocked list) and returns reward totals for ANY member — no check that the requested member matches the acting member or that the acting member is a parent. A child can query any other kid's or the owner's reward total. Low severity (reward totals are not highly sensitive), but it's a cross-member data leak. FIX: add the same check as `handleKidToday` (line 2381): child can only query their own member.

- **[LOW] CompanionServer.swift:612 — `/members` returns all household members' IDs, displayNames, and roles to any authenticated member including children.** This is by design (the kid surface needs the member list for the switcher), but it does leak the existence and role of all household members to a child. Acceptable for a family device, but worth noting.

- **[INFO] CompanionServer.swift:636-646 — `resolveMemberID` correctly validates the `X-hOS-Member` header against `members.member(id:)`.** Unknown IDs fall back to `owner` (line 645), which means a child cannot impersonate another member by sending a fake header — the header is resolved to a known member or defaults to owner. This is the correct pattern. The `makeAccessController(forActingAs:)` then resolves the role server-side. A child sending `X-hOS-Member: owner` would get owner's role ONLY if the header value matches owner's actual ID — but since the child doesn't know owner's ID... actually, `GET /members` returns all IDs. So a child CAN set `X-hOS-Member: <owner-id>` to impersonate the owner. However, this is a pre-existing design issue noted in prior reviews (B2: "member hardcoded to owner", "no per-user auth on companion app"). The companion app uses a single shared bearer token, not per-member auth. The `X-hOS-Member` header is a role-switching mechanism, not an auth mechanism. This is the documented beta design. The server-side role enforcement at line 535 relies on `access.acting.role` which is derived from the header — so a child who knows the owner's member ID can bypass ALL kid-surface restrictions by setting `X-hOS-Member: <owner-id>`. This is the fundamental weakness of the header-based member model.

- **[INFO] LLMService.swift:1279 — `autoRecall` correctly skipped for child.** `guard currentMemberRole != .child else { return }` is a hard server-side block on the automatic memory injection path. Solid.

- **[INFO] LLMService.swift:1230-1236 — `refreshSystemMessage` correctly swaps to kid-safe prompt for child.** No owner-scoped notes or directives injected. Solid.

- **[INFO] CompanionServer.swift:2381 — `/kid/today` child cross-member check is solid.** `if access.acting.role == .child && requestedMember != access.acting.id` → 403. Parents/owners can request any member (for monitoring). Correct.

- **[INFO] CompanionServer.swift:1062-1079 — `/loops/complete` chore path has correct assignee check.** `isAssignee || isParent` using `access.acting.role` (server-side). The HTTP path is solid; the LLM tool path (see MEDIUM above) is not.

### Patterns
- **Server-side role enforcement via `access.acting.role` is the gold standard.** Every HTTP handler that checks `access.acting.role` is correct. Every LLM tool handler that checks `arguments["member"]` or `currentMemberRole` instead is vulnerable. The lesson: LLM tool handlers MUST use `currentMemberRole` (set from `access.acting.role` in `reply(to:memberRole:)`), NOT client-influenced tool arguments.
- **`buildTools()` should be role-aware.** The tool list sent to the LLM is the API surface. If a tool is not in the list, the LLM cannot call it. Filtering `buildTools()` by `currentMemberRole` is the strongest defense against LLM-mediated data exfiltration — stronger than prompt-level guardrails.
- **The `X-hOS-Member` header is role-switching, not authentication.** Any holder of the shared bearer token can impersonate any member by knowing their ID (available via `GET /members`). This is the documented beta design but is the root cause of several role-bypass risks. Per-member authentication (e.g., per-member tokens or biometric gating) would close this class of issues.

### Postgres bundling review (feature/postgres-not-starting, 2026-08-22)
- **Bundled binaries must be codesigned with hardened runtime (--options runtime), not just ad-hoc signed.** bundle-postgres.sh line 179 uses `codesign --force --sign -` without `--options runtime`; Xcode's archive step may re-sign, but ad-hoc signing alone does not guarantee hardened runtime entitlements are applied to helper binaries, which is required for notarized distribution.
- **`git clone` in a build script pins to a tag (--branch v0.8.6) but does not verify a commit hash or GPG signature.** A compromised upstream tag or repo could inject malicious code into the pgvector build at build time. Pin to a commit SHA and verify it.
- **startupError string is interpolated from raw error descriptions and displayed in the Admin UI (AdminSurface.swift:900).** While the current error messages are controlled, Swift error descriptions can include file paths or internal details; sanitize before display to avoid leaking filesystem layout or internal state to anyone with Admin panel access.
- **pg_hba.conf uses `trust` for all local connections (PostgresManager.swift:319-321).** This is acceptable for a single-user daemon on loopback, but `trust` on `local all all` means any local OS process can connect without credentials. Consider `scram-sha-256` with a generated password stored in Keychain for defense-in-depth.
- **The postgres data directory path is derived from the app bundle identifier via FileManager.applicationSupportDirectory.** If the bundle ID is ever changed or the app is sandboxed differently, the data directory could resolve to an unexpected location. Verify the resolved path at first launch and log it.

## CloudKit sync diagnostics review (feature/approvals-not-syncing-ios, 2026-08-22)

- **Conflict-retry `try?` returning `true` is a false-success pattern.** `writeRecord` catches `.serverRecordChanged`, re-applies the payload to `ckErr.serverRecord`, then does `_ = try? await db.save(server)` and returns `true`. The `try?` swallows a retry-save failure, so the caller sees success even when the record was NOT persisted. When a diagnostic layer (new `diagnostics.lastOutboxWrite`/`lastOutboxWriteError`, `checkOutboxHealth`, `testSync`) trusts that return value, it reports "Syncing / OK" for a write that silently failed — the exact failure the feature exists to catch. FIX: capture the retry-save result; return `false` (and log the CKError) if the retry throws. Generalize: any `try?` on a persistence call whose result drives a success indicator is a fail-insecure bug.
- **Error-swallowing `fetchRecord` collapsing real errors into "not found".** `fetchRecord` returns `nil` for BOTH `.unknownItem` (genuinely absent) and any other `CKError` (network/quota/auth). Callers (`CloudApprovals.reload`, `CloudMailbox.sync`) then treat `nil` as "no record published yet" / `lastFetchResult = "notFound"`, masking a live fetch failure as a benign empty state. A transient CloudKit outage is reported as "no approvals pending" rather than "error." FIX: make `fetchRecord` tri-state (found / absent / error) or have it throw/return the error so reload can set `lastFetchResult = "error"`. The CloudMailbox code documents this gap in a comment but never resolves it.
- **Code-signing self-introspection (SecStaticCode) is safe — no TCC concern.** `SecStaticCodeCreateWithPath(Bundle.main.bundleURL)` + `SecCodeCopySigningInformation` reads the app's OWN signature to extract the `aps-environment` entitlement as a CloudKit Dev/Prod proxy. It is not inspecting other apps and needs no TCC entitlement. Correctly guarded by platform (`#if os(macOS)` in CloudApprovals, `#if canImport(Security)` in CloudMailbox).
- **`canImport(Security)` vs `os(macOS)` inconsistency is a portability smell.** `canImport(Security)` is true on iOS but the `SecStaticCode*` symbols are not exported there → compiler errors if the file is ever compiled for iOS. CloudMailbox.swift (Mac-only target) gets away with `canImport(Security)` today; CloudApprovals.swift correctly uses `#if os(macOS)`. Standardize on `#if os(macOS)` for code-signing query symbols regardless of current target, per the coder.md lesson.
- **No PII/credential exposure in os_log — good privacy discipline.** Logs contain only record names (static constants), item counts, byte sizes, CKError numeric codes + enum-case names, and account-status labels. Approval payload content (detail text, recipients, member names) is deliberately NOT logged — only `payloadBytes`. All interpolated values use `.public` privacy explicitly where the value is non-sensitive, and `error.localizedDescription` (Apple-controlled, not user-derived) is the only dynamic error text. CloudKit record names are static constants, not user input, so there is no path-traversal/injection surface into `CKRecord.ID(recordName:)`.
- **Probe-record cleanup hygiene (non-security).** `testSync()` returns early on readback failure BEFORE calling `deleteRecord`, leaving the `hos-approvals-test-sync` marker lingering. It is overwritten on the next probe and is not sensitive, but the delete should happen in a `defer`/finally for tidiness. `checkOutboxHealth` correctly deletes unconditionally after its probe.
- **No privilege-escalation surface in the new writes.** All writes target `privateCloudDatabase` of the signed-in iCloud account under fixed record names; the conflict path overwrites a server record authored by the same device. The shared single-record outbox (all members' approvals in one record) is a pre-existing design noted in prior reviews, not introduced or worsened here.

---

## Event-Driven Pipeline: Mandatory Final Steps

**These calls are mandatory when running under the event-driven pipeline (HOS_TASK_GID is set).**

On PASS (no BLOCK-level findings) — call as your absolute last action:
```bash
~/ADTools/skills/hos-pipeline-trigger/trigger.sh \
  --task-gid "$HOS_TASK_GID" \
  --branch "$HOS_BRANCH" \
  --scope-doc "$HOS_SCOPE_DOC" \
  --completed-step reviewer-security \
  --next-step docs \
  --summary "[one-line result, e.g. Security APPROVED: no path traversal, injection, or credential exposure]"
```

On BLOCK (BLOCK-level security finding) — call before exiting:
```bash
~/ADTools/skills/hos-pipeline-trigger/fail.sh \
  --task-gid "$HOS_TASK_GID" \
  --branch "$HOS_BRANCH" \
  --failed-step reviewer-security \
  --reason "[specific finding, e.g. Path traversal in SkillHost.swift:88 — unsanitized user input in file path]" \
  --needed "[what fix is required, e.g. sanitize the input with explicit path component validation before passing to FileManager]"
```

**If HOS_TASK_GID is not set** (old cron-driven invocation): behave as before — no trigger or fail call needed.

Pipeline envelope variables injected by trigger.sh: HOS_TASK_GID, HOS_BRANCH, HOS_SCOPE_DOC, HOS_COMPLETED_STEP, HOS_NEXT_STEP, HOS_SUMMARY, HOS_STARTED_AT.
- 2026-09-07 (postgres lifecycle rework r4 re-review, security): Fail-closed default verified end-to-end — `pidFileFailedValidation = true` plus allow-list `noPidFileExisted && classifyPort == .free` preserves the invariant (pg_ctl never runs off an unvalidated pid file); residual risk on the fresh-start gate (an unvalidated pid file appearing in the TOCTOU window before pg_ctl reads line 1) is bounded to pg_ctl launch latency and only reachable in the same-user fresh-start scenario — LOW, consistent with the accepted r2/r3 TOCTOU framing.
