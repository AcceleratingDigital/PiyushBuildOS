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
