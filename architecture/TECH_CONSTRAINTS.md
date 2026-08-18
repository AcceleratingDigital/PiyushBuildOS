# hOS Technical Constraints & Architecture

> Living document. Updated as architectural decisions are made.

## Infrastructure Stack
- **Server:** Single supervised macOS process using Swift/Vapor.
- **Database:** Postgres + pgvector (sidecar) for memory, loops, and audit.
- **Inference:** Local-first via LiteLLM with a Model Router managing fallback
  chains to cloud providers and on-device (Apple Foundation Models).
- **Client:** SwiftUI companion app (iOS/macOS) using hOSKit shared contracts.
- **Transport:** Bonjour (home), APNs + CloudKit mailbox (remote v1),
  owner-relay (remote v2).

## Hard Constraints
- **Local-First:** No user data stored on external servers by default.
- **TCC/Privacy:** Single platform-level permission story for the server process.
- **Model Agnostic:** Feature specs must never reference specific models;
  selection is a runtime decision by the Model Router.
- **Server-First Intelligence:** Heavy synthesis on server; lightweight
  interaction/approval on the edge (iPhone).

## Key Design Patterns
- **Gateway Chokepoint:** Every tool call passes through the Vapor gateway for
  auth, scope check, and audit.
- **Shared Capabilities:** Common logic (summarization, entity extraction) must
  be implemented as shared services to avoid per-skill redundancy.
- **Git is Source of Truth:** For build status; Asana is source of truth for
  product intent.

## Model Selection (build-time, not spec-time)
See `~/.hermes/skills/devops/agent-operation-directives/references/model-selection-preferences.md`
for the full model-to-task mapping. Summary:
- Logic/code/review/coordination: `algolia/xlarge` via LiteLLM
- QA verification: `algolia/medium` via LiteLLM
- HTML/CSS/design work: `claude-sonnet-4-6` via LiteLLM
- Tech docs: `claude-sonnet-4-6` via LiteLLM
- Deep security review: `gpt-5.6-sol` (OpenAI, for model diversity)

## hOS Data-Access Performance Facts
| Data | Fastest access | Why | Example skill |
|---|---|---|---|
| Email (Mail.app) | SQLite (`Envelope Index`) | 10-100x faster than JXA | EmailRead |
| Calendar | EventKit (Swift) | Native framework | CalendarRead |
| Contacts | Contacts framework (Swift) | Native framework | ContactsSearch |
| Messages | SQLite (`chat.db`) | Direct DB access | MessagesRead |
| Notes | JXA (osascript) | No framework, only option | NotesRead |
| Reminders | EventKit (Swift) | Native framework | RemindersWrite |
| Spotlight search | `mdfind` (Process) | Indexed search | SpotlightSearch |
| Obsidian/journal | FileManager | Plain markdown files | JournalRead |
| Finance | SQLite (hOS ledger) | Internal hOS DB | FinanceQuery |

**JXA is slow.** Only use when no faster path exists (Notes is the only case).
