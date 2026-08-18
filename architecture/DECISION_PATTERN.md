# hOS — Decisions Needed Before Build-Out

> August 15, 2026. Comprehensive analysis of 120 Asana tasks, 39 research topics,
> 21 competing products, and all hOS design docs. Each item below blocks forward
> progress — on product scope, architecture, sequencing, or competitive positioning.
> Answer each decision; the answers determine the build plan.
>
> **Spec principle (established):** Feature specs must NOT specify which model is used.
> Model selection is a build/runtime decision owned by the Model Router, invisible to
> users and specs. Specs use three sections: **What it does** (user behavior),
> **How it works** (architecture/flow), **What it can use** (shared capabilities, APIs,
> data sources, other skills via subSkillCalls).

---

## A. Product Scope Decisions — "What IS the beta?"

### A1. Beta scope: vertical slice vs. horizontal sweep

**The tension:** 0.9 scope builds 12 platform foundations (install, multi-member reads,
approval, write skills, vault, model router, memory, admin surface). But the vision's
success criteria requires *daily use replacing inbox-checking* — that needs the Daily
Brief + Open Loops working end-to-end, not just platform plumbing.

**Decision needed:** Is the August 21 beta:
- **(a) Platform foundation only** — install works, reads work, approval works, but no
  Daily Brief or Open Loops yet. Testers validate mechanics, not value.
- **(b) One complete vertical** — Mail triage → Daily Brief → approval → loop queue,
  end-to-end. Testers experience the value prop on day 1. Finance and other domains
  come later.
- **(c) Both verticals** (Mail + Finance) as the vision originally specified — higher
  risk, broader test.

**Competitive pressure:** Every product researched (Shortwave, Superhuman, Copilot
Money, Monarch) wins on a complete vertical experience. A beta that only proves
plumbing won't convince anyone to switch. **Recommend (b)** — one complete vertical
(Mail) that demonstrates the full hOS loop.

**Answer:** **(b) One complete vertical (Mail)** — Mail triage → Daily Brief → approval → loop queue, end-to-end. Finance added as second vertical (see A4). Decided Aug 16, 2026.


### A2. Open Loops / Unified Queue — build for beta or defer?

**The gap:** The vision calls Open Loops "the core hOS primitive" and a v1 success
criterion. It's tagged for beta (due Aug 21). But there's no design doc, no spec, no
subtasks — just a task name. This is the biggest gap between vision and plan.

**Decision needed:** Is Open Loops a beta blocker? If yes, it needs a design spike NOW
— what is a loop, how does it differ from a task, how does it surface in the UI, how
does it relate to the Daily Brief? If no, the beta ships without the core primitive and
testers see "just another assistant."

**Competitive pressure:** Skylight, Cozi, and every family organizer have shared lists.
Shortwave and Superhuman have inbox-as-queue. hOS's differentiator is that loops are
*agent-managed* (the agent creates, tracks, and closes loops), not just human lists.
Without this, hOS is a read+draft assistant, not a Chief of Staff.

**Answer:** **(a) Build full Open Loops for beta** — Open Loops is the core differentiator. Agent-managed loops (agent creates, tracks, closes), not human lists. Needs design spike + build. Decided Aug 16, 2026.


### A3. Daily Brief — composite skill or platform feature?

**The gap:** Morning Status / Daily Brief is tagged for beta. It's a composite skill
(calendar + email + messages → brief). But the 0.9 scope doesn't list it as a build
item — it lists "morning-system-status composite" as a read skill to port, which is
different.

**Decision needed:** Is the Daily Brief for beta a simple "read calendar + read email →
text summary" or the full vision (agent-synthesized brief with prioritized loops,
proactive alerts, weather, meal plan)? The simple version is buildable in days; the
full version requires Open Loops + agent loop maturity.

**Answer:** **(c) Full agent-synthesized brief** — The Agent Loop itself produces the brief, not Morning Status. Agent reviews all data sources, decides what matters, creates loops, drafts responses, presents synthesized brief. Morning Status becomes a data source the agent queries. Decided Aug 16, 2026.


### A4. Finance in beta or not?

**The gap:** Finance is tagged for beta (Transaction Ingestion, Reporting, Anomaly
Alerts — all due Aug 21). But D-05.3 says "no transaction initiation in Phase 1" and
finance needs SimpleFIN token + hosted bank-definition pipeline (your infra). The 0.9
scope lists finance under "NEEDS YOU (creds / infra)."

**Decision needed:** Is finance a real beta feature or a stub? If beta, you need to
provide the SimpleFIN token and confirm the bank-definition pipeline is ready. If stub,
remove the Aug 21 due dates from finance tasks.

**Competitive pressure:** Copilot Money, Monarch, YNAB, and Rocket Money all have mature
finance experiences. hOS's beta finance needs to at minimum match "read transactions →
categorize → alert on anomalies" or it's not worth switching to.

**Answer:** **(b) Finance as second vertical** — Finance Import + Query already shipped (v0.4.0). Delta: categorization + anomaly detection + agent integration. Agent triages imports, creates loops for anomalies, surfaces in Daily Brief. Needs SimpleFIN credentials or CSV test data from Piyush. Decided Aug 16, 2026.


### A5. Companion iOS app — beta blocker?

**The gap:** Companion iOS App is tagged for beta. The 0.9 scope says "needs Apple Push
cert + a big new target. Decision: this round or next." This is unresolved.

**Decision needed:** Does the beta require the iOS companion, or can testers use the
macOS app + web fallback? If iOS is required, the Apple Developer Program enrollment
($99, D-05.15) needs to happen immediately and APNs cert setup begins.

**Competitive pressure:** Every family organizer (Cozi, FamCal, Skylight) is
mobile-first. Testers won't sit at a Mac to approve things. If iOS isn't in beta, the
approval flow — hOS's core trust mechanism — can't be tested in real life.

**Answer:** **(a) Full 5-tab iOS app for beta** — Today (Daily Brief + blocking approvals + due-soon loops), Loops (grouped by domain, timeline, drafts), Approvals (two sections: Needs you now + Work queue), Chat (conversation with agent), Me (personal agent directives, brief settings, whitelist rules, connected accounts, agent audit, memory view). 

**Add-on: Personal Agent Directives** — Members define, disable, and delete rules that govern hOS behavior for them personally (brief timing/content, update periods, whitelist rules). Stored in member's own memory scope. Managed from Me tab. Per-member, independent of other members. No agent-suggested directives or scoped directives for beta — just member-created, member-managed. Decided Aug 16, 2026.


### A6. Onboarding — how much for beta?

**The gap:** Onboarding is tagged for beta. D-05.16 names Household 2 (Metin + Jes) as
the "stranger install test." The vision says "fresh install on clean Mac mini in under
an hour."

**Decision needed:** Is the beta onboarding (a) you manually set up each tester's
machine, or (b) a self-service installer that Metin/Jes can run without you present? If
(b), onboarding needs to be a real build item with member enrollment, device pairing,
and integration setup.

**Answer:** **(b) Self-service installer** — Guided first-launch wizard: LLM config → TCC permissions → member enrollment → account connections → verify. Step-by-step with error recovery. Testers (Metin/Jes) self-onboard without Piyush present. Proves "fresh install in under an hour" claim. Decided Aug 16, 2026.


---

## B. Architecture Decisions — "How do we avoid refactoring later?"

### B1. Shared Capabilities Architecture — WHEN to design?

**The gap:** Research task (high priority) created this session. hOS has
`subSkillCalls`/`providesTo` in the skill standard, but no process step enforces reuse.
Summarization, entity extraction, KB write/read — all will be reinvented by every skill
unless the shared-capability contract is defined first.

**Decision needed:** Design the shared-capability registry + process enforcement BEFORE
building any skill that needs summarization/extraction (which is most of them). If we
don't, every skill (Mail, Finance, Journal, Podcast pipeline) will have its own
summarizer and we'll refactor all of them later. **This is the highest-leverage early
decision.**

**Competitive pressure:** Apple Intelligence's advantage is system-wide capabilities
(summarization, entity recognition, semantic search) that every app uses. Khoj and mem0
both built shared memory/extraction layers. hOS needs the same.

**Answer:** **(a) Design now, before any content skill** — Full capability registry + subSkillCalls runtime + shared summarize/extract/categorize/KB interface. First build item in the beta plan. Blocks all content-processing skills. Decided Aug 16, 2026.


### B2. Memory v1 (SQLite) vs. Memory v2 (pgvector) — which for beta?

**The gap:** 0.9 scope says "Memory v1 (four-layer, D10), local store." But Memory v2
(pgvector) is a separate Asana task. The Universal Knowledge Trigger and
Podcast/Newsletter pipeline both depend on semantic recall, which needs pgvector.

**Decision needed:** Does beta ship with SQLite memory (simpler, faster to build, no
semantic recall) or pgvector (enables knowledge base, RAG, semantic search, but needs
Postgres setup)? If SQLite, the knowledge features can't be tested in beta. If
pgvector, more infra setup but the differentiating features work.

**Competitive pressure:** Khoj, mem0, Obsidian, and Logseq all have semantic search.
Without pgvector, hOS's "second brain" is keyword-only — not competitive.

**Answer:** **REVISED: Bundle Postgres+pgvector now** — Apple's system SQLite cannot load extensions (`sqlite3_enable_load_extension` is stripped), so any SQLite-based vector solution is throwaway. Postgres is a planned D13 sidecar — bringing it forward, not inventing new architecture. Bundled binary (~50MB, arm64), pgvector compiled in, hOS manages lifecycle as child process. NLEmbedding (512-dim) as embedding fallback when Ollama unavailable. This supersedes the original (d) answer. Decided Aug 16, 2026 (revised from initial (d) after SQLite extension testing).


### B3. Universal Knowledge Trigger — platform pattern or feature?

**The gap:** Research task created this session. Every processed item (email,
transaction, journal, note, message, calendar, podcast, newsletter) could feed a
knowledge base. This is a cross-cutting architectural pattern, not a single feature.

**Decision needed:** Is the KB trigger contract part of the beta platform (so all skills
emit knowledge entities from day 1), or is it a post-beta addition that requires
retrofitting skills? If post-beta, skills built for beta won't have the trigger and will
need updates.

**Answer:** **(a) KB trigger as platform pattern — all skills emit knowledge from day 1** — KnowledgeEmission protocol in SkillKit. Every skill calls context.emitKnowledge(entities:). Shared Capabilities extract produces entities. MemoryStore stores with NLEmbedding vectors. All skills updated for beta. Decided Aug 16, 2026.


### B4. Per-member macOS sessions — still the plan?

**The gap:** D-05.2 said "each member gets a macOS login on the server." But this was
"reopened 2026-08-13" because the server is now Swift single-process. D-05.16 still
references multi-member hosting. The 0.9 scope item #2 says "Multi-member reads —
complete" but the mechanism is unclear.

**Decision needed:** How does the beta handle multi-member? Is it (a) one macOS session
with the server reading each member's data via helper/protocol, or (b) Fast User
Switching with per-member sessions? This affects TCC permissions, data access, and the
entire multi-member architecture.

**Answer:** **(a) Keep current — root FDA helper** — Already built, already works. D4 experiment validated one process, TCC once, serve all members. Owner data in-process, other members via root helper. No change for beta. Decided Aug 16, 2026.


### B5. Process Isolation (XPC) — when?

**The gap:** LEARNINGS doc says in-process skills are "a policy boundary, not a
sandbox." XPC is needed for untrusted/third-party skills. Not in 0.9 scope.

**Decision needed:** For beta, are all skills in-process (trusted, first-party only) or
do we need XPC before beta? If beta testers only use first-party skills, XPC can wait.
If beta includes any third-party or community skills, XPC is a blocker.

**Answer:** **(a) All in-process for beta, XPC deferred to v2** — All 15 beta skills are first-party, signed by our team, Hardened Runtime library validation already blocks unsigned bundles. No third-party/community skills in beta. XPC design stays documented as Future; build it when an actual third-party skill needs it. Decided Aug 16, 2026.

### B6. Contract Versioning — before auto-update ships?

**The gap:** Tagged for beta (due Aug 21). LEARNINGS doc says manifests should carry
SkillKit version. Sparkle auto-update is in 0.9 scope. If auto-update ships without
contract versioning, an update could silently break installed skills.

**Decision needed:** Is contract versioning a beta blocker for the auto-update feature?
If yes, it must be built before Sparkle ships. If no, auto-update could break skills on
update and testers lose trust.

**Answer:** **(a) No contract versioning for beta, defer to v2** — All beta skills ship in PlugIns (atomic with Sparkle update). No locally-deployed extras in beta. ABI-skew fails gracefully (load-log, not crash). Version handshake built in v2 alongside XPC when third-party skills need it. Decided Aug 16, 2026.

---

## C. Competitive Positioning — "Why switch from what I use today?"

### C1. What replaces each current tool?

People don't adopt new tools — they switch. For each beta tester, hOS needs to be
visibly better than what they already use:

| Current tool | What hOS must beat | Risk |
|---|---|---|
| **Apple Mail + flags** | Mail triage with AI summarization, draft replies, loop creation | Shortwave/Superhuman set the bar high |
| **Calendar app** | Calendar read + conflict detection + draft events | Calendar apps are already good — hOS needs the agent layer (proactive) |
| **Notes app** | Notes read + semantic search + KB integration | Apple Notes is simple but ubiquitous |
| **Reminders** | Reminders via bridge, mutate-approved | Already works on Apple ecosystem |
| **Cozi/Skylight (families)** | Daily Brief + shared calendar + chore charts + meal plan | Skylight's wall display is a killer feature hOS doesn't have yet |
| **Copilot/Monarch (finance)** | Transaction ingestion + categorization + anomaly alerts | These are mature products — hOS finance is read-only |
| **Find My** | Location sharing + location triggers | Apple Find My is already on every device |

**Decision needed:** For each beta tester, which specific tool does hOS replace? If the
answer is "none completely," what's the wedge that gets them to start using hOS
alongside their current tools? The vision says "daily brief + loop queue replaces
inbox-checking" — that's the wedge. But it requires A1(b) and A2.

**Answer:** **(a) Alongside positioning** — hOS doesn't replace any tool for beta. It replaces the *behavior*: stop checking 5 apps 20x/day, check your Daily Brief + Open Loops instead. hOS is an agent layer on top of Mail, Calendar, Notes — triages, synthesizes, proactively manages. Beta testers keep their existing tools. Wedge requires A1(b) Mail vertical + A2 Open Loops. Decided Aug 16, 2026.

### C2. The Skylight problem — shared family display

**The gap:** Skylight's wall-mounted calendar display is the product families actually
buy and use. We designed an iPad Ambient Surface mockup, but it's in Research Topics,
not beta scope. Cozi, FamilyWall, FamCal all have shared family views.

**Decision needed:** Is the iPad Ambient Surface a beta feature? If not, families using
Skylight have no reason to switch — hOS is "just" a per-member assistant on their
phones. The shared household surface (iPad or otherwise) may be the key differentiator
vs. per-member assistants (Siri, Gemini, Alexa).

**Answer:** **(b) Minimal shared surface for beta — "Today" view on iPad** — The iOS app's Today tab (Daily Brief + household calendar + open loops) adapts to iPad layout. Not a full ambient display — just the app running on iPad in shared context. "Put the iPad on the counter and open hOS." Low effort (SwiftUI adapts naturally), gives families a shared view to learn from. Purpose-built ambient display deferred to v2. Decided Aug 16, 2026.

### C3. Chores & Kids Tasks — table-stakes for families?

**The gap:** Identified as a true gap. Skylight's star-powered chore charts are their
killer feature. hOS has no kids task surface. Doc 11 mentions "kid variant" of the
member surface but no chore/reward system.

**Decision needed:** Is chore tracking a beta feature or post-beta? If your household
includes Sophie (kid member, D-05.16), chores could be tested immediately. But no
design exists.

**Answer:** **(b) Lightweight chores as Open Loops extension** — Chores are a special loop type (kind: chore) with assignee, reward, recurrence. Parent assigns, kid completes, parent approves, reward tracking. Kid sees chores in Loops tab filtered to their scope. iPad shared surface gets a simple chore chart view. Reuses Open Loops infrastructure (A2 dependency). Improve to full system in v2. Decided Aug 16, 2026.

### C4. Advanced Email AI — parity with Shortwave/Superhuman?

**The gap:** hOS Mail is basic read/triage/draft. Shortwave has AI filters in plain
English, 3-tier intelligence, bundles. Superhuman has instant reply, follow-up tracking.
SaneBox has learning, daily digest.

**Decision needed:** For beta, is hOS Mail (a) basic triage + summarize + draft (matches
what 0.9 scope implies), or (b) AI-native email with smart filters, snooze, bundles,
daily digest? (a) is buildable; (b) is a major feature area. If (a), hOS Mail is behind
Shortwave/Superhuman on day 1 — is that acceptable for beta?

**Answer:** **(a) Agent-layer triage with training/learning loop — NOT an email client** — User continues working in their email client (Apple Mail) on their device. hOS's job is to create loops and learning ability: observe, classify, flag, extract intelligence, create Open Loops for action items. Over time, hOS learns patterns from user behavior (read/delete/reply) and proposes automation rules. User trains hOS by accepting patterns into memory. Things become more "superhuman-like" but ONLY after the user has defined a pattern and accepted it — not from day 1. No smart filters, snooze, bundles, or inbox UI. hOS is an agent layer, not a mail client. Reference implementation: ADTools inbox-scanner V1+V2 (428 rules, behavioral observation, AGENT: notes for exceptions, graduation from observed patterns to auto-action). Decided Aug 16, 2026.

**Key design principles from ADTools reference system:**
- Observe, don't command — watch what user does in Mail.app, learn from it
- No auto-delete, ever — most aggressive auto-action is auto-mark-read
- Human-in-the-loop via Open Loops (not Telegram commands) — user reviews flagged emails as loops, adds instructions, approves
- 5 items at a time for email review — don't overwhelm
- AGENT: notes are for exceptions, not training — high friction, target ≤5/week
- Learning loop: detect behavioral transitions (read/delete/reply) → aggregate patterns per sender+subject → propose graduation (auto-action) → user approves → hOS auto-acts
- Intelligence extraction: due dates, dollar amounts, tracking numbers, confirmation codes → stored in memory for recall
- Things become superhuman-like over time as patterns graduate to automation, not from building email UI features

### C5. Meal Planning — core family feature or future?

**The gap:** 4+ products (Skylight, Cozi, FamCal, Alexa) treat meal planning as core.
hOS has it buried as "future" in Knowledge & Media. The coverage audit elevated it.

**Decision needed:** Is meal planning (recipe → calendar → grocery list loop) a beta
feature? It's a high-visibility family feature that could differentiate hOS from
per-member assistants. But it requires recipe storage, grocery list management, and
calendar integration — non-trivial.

**Answer:** **(b) Lightweight meal planning for beta — agent-driven via Open Loops + memory** — "Plan meals for the week" → agent suggests based on household preferences (memory) → creates calendar events for meals → generates grocery list as an Open Loop. No recipe database or grocery list UI — it's the agent applying triage/draft/loop-creation to meals. Reuses Open Loops (grocery list = a loop), memory (preferences, past meals), calendar (meal events). iPad shared surface shows meal plan. Low effort if agent loop works. Having a baseline to try and give input on is more valuable than waiting for full specs. Expand to full capability in v2. Decided Aug 16, 2026.

---

## D. Process & Sequencing — "What order do we build?"

### D1. Shared Capabilities before any content-processing skill

**Decision needed:** Build the Shared Capabilities Architecture (B1) FIRST, before Mail
summarization, Finance categorization, Journal capture, or Podcast pipeline. If we
build skills first, each one reinvents summarization/extraction and we refactor later.
This is a sequencing decision, not a scope decision.

**Answer:** **(a) Shared Capabilities first (confirms B1)** — Build platform-level summarization/extraction/classification before any content skill. Avoids ADTools failure mode where each skill reinvents its own approach. Foundation is now concrete: Postgres+pgvector for storage, managed Ollama for inference, model manifest for tier selection. Content skills (Mail, Finance) built on top immediately after. Decided Aug 16, 2026.

### D2. Research → Ready to Plan → Build pipeline

**The gap:** 39 research topics exist. 0 are in Ready to Plan. No items have moved
through the pipeline yet. The process (research → ready to plan → build) was defined
this session but hasn't been exercised.

**Decision needed:** Which research topics are ready to move to "Ready to Plan" now? The
Shared Capabilities Architecture is the most urgent. Others (iPad surface, chore charts,
podcast pipeline) need more research before planning. We need to start moving items
through the pipeline.

**Answer:** **(a) Move the 8 ready items to Ready to Plan now** — Kick-start the pipeline. Items: Shared Capabilities, Managed Ollama, Postgres+pgvector sidecar, Semantic Memory Layer, Open Loops/Unified Queue, Agent Loop/Model Router, Mail triage with learning loop, Finance vertical. Start speccing immediately. Items needing more research (iPad surface, chores, meal planning, podcast pipeline) stay in Research Topics. Decided Aug 16, 2026.

### D3. Beta scope reconciliation — 32 tasks tagged, but are they all real?

**The gap:** 32 tasks are tagged for beta (Aug 21, assigned to you). But some of those
tasks are platform foundations (Agent Loop, Storage, Vault) while others are
user-facing features (Mail, Calendar, Notes). Some are tagged but have no design doc
or spec. Some (like Cross-household/Second Install) require external testers. The Aug
21 date may be unrealistic for all 32.

**Decision needed:** Review the 32 beta-tagged tasks. Which are truly required for beta
(must block Aug 21)? Which are "nice to have" (can slip)? Which are mis-tagged (should
be post-beta)? The beta scope needs to be a coherent set, not everything tagged with
the same date.

**Analysis (prepared by requirements agent):**

Given A1-A5 (Mail vertical, Open Loops, agent brief, Finance second, full iOS app) and
D1-D2 (Shared Capabilities first, move 8 items to Ready to Plan), the beta scope is now
concrete. The 32 beta-tagged tasks need reconciliation against A1-C5:

**MUST HAVE (platform foundations for A1-A5 to work, 14 tasks):**
- Agent Loop / Model Router maturity (queuing, guardrails, metering)
- Shared Capabilities Architecture (extraction, summarization, classification)
- Memory Layer (SQLite+NLEmbedding per B2, not pgvector)
- Open Loops / Unified Queue (core to A2 and A3)
- iOS Companion App (5 tabs per A5)
- Credential Vault (per-member secret scoping, keychain-backed)
- Admin Surface (member management, audit, enable/disable)
- Mail Skill (triage, summarization, draft per A1)
- Calendar Skill (read, conflict detection per A1)
- Finance Skills (import, query, categorization, anomaly detection per A4)
- Personal Agent Directives (Me tab, member-defined rules per A5)
- Self-service Installer (per A6)
- Error Logging & Diagnostics (for beta feedback loop)
- Managed Ollama lifecycle (new research item, enables inference)

**SHOULD HAVE (features that enhance the verticals, 7 tasks):**
- Chore tracking as loop extension (C3, optional but low-friction)
- Meal planning agent-driven via loops (C5, optional but visible to families)
- iPad Today view adaptation (C2, SwiftUI layout of existing view)
- Finance anomaly detection refinement (C4, agent observes only at start)
- Journal skill read (part of Daily Brief data sources per A3)
- Notes skill read + semantic search (part of Daily Brief per A3)
- Messages skill read (part of Daily Brief per A3)

**MAYBE HAVE (often tagged beta but not required by decisions A1-C5, can slip, 11 tasks):**
- Cross-household / Second Install (D-05.16 names one household, one install for now)
- Advanced email AI filters/snooze/bundles (C4 explicitly: agent observes, no email client)
- Full Outlook/Gmail integration (A1 chose Mail primary for beta)
- Podcast/Newsletter pipeline (depends on Universal KB Trigger, post-beta per G3)
- Whitelist / Autonomy Growth (F2 open, not required by A1-C5)
- Per-member Delegation (F3, post-beta)
- Contract Versioning (B6 decided no, deferred to v2)
- Sparkle auto-update (related to B6, can ship v1 without it)
- Knowledge Dashboard (part of G5 elevation, post-beta)
- Research backlog items (8-10 tasks in Research Topics that aren't moving to Ready to Plan yet)

**Recommendation:** 
(a) **Tier the 32 into Build Now (14), Build Next (7), Defer (11)** — Prevents Aug 21 slip
by being clear about what's actually required. The 14 Must-Have tasks are what A1-C5
actually need. The 7 Should-Have tasks are enhancements that reuse core infrastructure
(loops, memory, agent). The 11 Maybe-Have tasks are nice-to-have but not blockers.

(b) **Move Defer tasks to v2 tags**, not Aug 21 — gives clear visibility that Podcast,
Delegation, Email Filters, etc. are legitimate post-beta features, not failures.

(c) **Keep Aug 21 only for Must+Should tiers** — 21 tasks. More realistic than 32.

**Answer:** **(a) Tier the 32 into Build Now (22), Build Next (1), Defer (9)** — All 14 Must-Have items locked (Agent Loop, Shared Capabilities, Memory, Open Loops, iOS, Vault, Admin, Mail, Calendar, Finance, Directives, Installer, Diagnostics, Ollama). All 7 Should-Have items moved to Must (Chores, Meal Planning, iPad, Finance Anomaly, Journal, Notes, Messages). Sparkle moved to Must. Advanced Email Filters moved to Should. Total: 22 Must for Aug 21, 1 Should, 9 Defer. Decided Aug 16, 2026.

### D4. Feature spec structure — standardize before writing more

**The gap:** Research notes in Asana are unstructured — some are feature ideas, some are
competitive analysis, some are architecture concerns. There's no standard spec format.
The spec principle (what it does / how it works / what it can use, no model references)
was just defined but hasn't been applied to existing notes.

**Decision needed:** Adopt the three-section spec structure for all features that move
from Research Topics to Ready to Plan. Research notes stay freeform; specs use the
structure. This ensures every planned feature has a consistent format before build
starts.

**Answer:** **(a) Three-section spec for all planned features** — All tasks moving from Research Topics to Ready to Plan carry the standardized spec in Asana notes:

```
## WHAT
[User-facing behavior, UI layout, data model, success criteria]

## HOW
[Implementation: files to create/modify, flow/logic, patterns, references to existing code]

## Build Readiness
- Dependencies: [what must ship first, or "none"]
- Parallel-safe: [yes/no and reasoning]
- Risk level: [low/medium/high]
- Risk reason: [why]
- Estimated scope: [S/M/L]
- Branch: feature/{slug}
```

Research notes stay freeform (competitor notes, rough ideas, product audit findings). Specs use structure. When an item moves to Ready to Plan (section move IS the signal), the spec is already in notes. Scope doc (docs/scope/{slug}.md) goes on the feature branch. Decided Aug 16, 2026.

---

## E. Risk & Trust — \"What can go wrong?\"

### E1. Beta tester trust — what happens when the agent makes a mistake?


---

## E. Risk & Trust — "What can go wrong?"

### E1. Beta tester trust — what happens when the agent makes a mistake?

**The gap:** The approval flow means no mutation happens without consent. But the agent
could still misread an email, miscategorize a transaction, or surface irrelevant
information in a brief. First impression matters — if the beta agent is wrong about
something important on day 1, trust is hard to rebuild.

**Decision needed:** What's the trust strategy for beta? Options: (a) conservative
defaults — agent only reads and summarizes, no proactive actions, (b) sandboxed
mistakes — agent can be wrong but mistakes are visible and correctable in the UI, (c)
full trust — let the agent work and see what breaks. Also: is there an undo for agent
actions?

**Analysis (prepared by requirements agent):**

Given A1-C5 decisions (Mail vertical with agent triage, Finance anomaly detection, agent-synthesized brief), the agent WILL make mistakes. A pure "never trust" stance (option a) defeats the beta's purpose — testing the agent loop maturity. Option (c) "full trust" is too risky for first beta impressions. Option (b) is the right fit.

Key trust mechanics already built:
- Approval flow: no mutations without consent (A1, A5)
- Agent observation only at start: agent reads mail, flags patterns, proposes automation to user via Open Loops (C4 reference: ADTools inbox-scanner, behavioral observation only)
- Learning loop: user teaches agent patterns over time, automation graduates from observed behavior (not day-1)
- Loops as the UI for corrections: user can review, edit, reject loops before approving (A2, A3)

**Design for beta:**
- Agent reads and synthesizes (high visibility, easy to correct)
- Agent flags items for human review (no auto-mutations)
- Mistakes are visible in loops/approvals (user can see and fix)
- No undo needed because agent never acted without approval
- Testers give feedback on what the agent got wrong → data for post-beta improvements
- Win metric for trust: "I caught a potential mistake the agent flagged, and I fixed it" (gives user agency)

**Answer:** **(b) Sandboxed mistakes with visible corrections** — The agent is allowed to misread, misclassify, or surface irrelevant info in the Daily Brief and Open Loops, because: (1) user sees it all before anything happens (loops are draft-state), (2) user can correct/delete/adjust before approving, (3) mistakes become training data for the learning loop (C4). No undo needed because approval is the point where action happens. Beta testers expect to help shape the agent — mistakes are expected and valued as calibration data. Decided Aug 16, 2026.


### E2. Data loss — what if the beta database corrupts?

**The gap:** Beta testers' data (mail history, finance ledger, memory, loops) lives in
the hOS database. If it corrupts, that data may not be recoverable from source systems
(mail can be re-read, but categorized/summarized/trained data is lost).

**Decision needed:** What's the backup strategy for beta? Is there automated backup
before we let testers in? Is there a "export my data" path? D-05.20 says jobs are
durable and cancellable, but that's job state, not data backup.

**Analysis (prepared by requirements agent):**

Beta is a test environment for 4 people (Piyush + 3 testers from Metin/Jes household + family) on a local-to-network hOS server. Data sources are recoverable (mail from Mail.app, transactions from SimpleFIN, calendar from Calendar.app, contacts from Contacts.app), except:
- Agent-learned patterns (C4 learning loop) — trained on observation, not recoverable
- User-created loops (A2, A3) — original creations, not recoverable
- Personal Agent Directives (A5) — custom rules, not recoverable
- Memory layer (semantic + structured) — curated by agent + user, partially recoverable if queries re-run

For a 4-person beta, manual recovery is acceptable. The testability of backup/restore itself is lower priority than shipping the core product (Mail vertical, agent loop, approval flow).

**Strategy for beta:**
- Automated daily backup (SQL dump) to local storage, kept for 7 days
- Manual export available for full member data (queries to JSON) — helps testers understand their data
- On corruption: restore from last backup, notify tester, accept learned-pattern loss as data test risk
- No automated cloud backup (keeps infrastructure simple for beta, no AWS/GCS dependency)
- Post-beta: add cloud backup or WAL-based replication if multi-household deployment is planned

**Answer:** **(a) Local automated daily backup, manual export path, accept learned-pattern loss** — hOS backs up the SQLite database nightly (7-day retention) to local storage. Members can export their data (all queries) as JSON for inspection/backup. On corruption, restore from backup, notify tester, re-run history if needed. Learned patterns and user-created loops will be lost (accepted as beta test risk). No external cloud storage for beta. Post-beta adds cloud replication if multi-tenant deployment is planned. Decided Aug 16, 2026.


### E3. Privacy boundary — what can the admin see?

**The gap:** D-05.19 defines superadmin authority with notice, delay, and audit. But
for beta, you (Piyush) are the admin AND a member. Can you see other members' private
data? Sophie's? Metin's? The privacy model needs to be clear before testers trust the
system.

**Decision needed:** For beta, what's the admin visibility boundary? Can the admin see
all member data by default, or is per-member privacy enforced even against the admin?
D-05.19 allows override with notice/delay — is that active for beta?

**Analysis (prepared by requirements agent):**

D-05.19 (Admin Authority) defines: superadmin can override with documented reason + delay + audit trail. For beta:
- 4 members (Piyush as owner + 3 household members)
- Piyush is simultaneously admin AND member
- Metin/Jes/Sophie are first testers from outside Piyush's household
- Sophie is a minor (requires parental oversight from Metin/Jes, not Piyush)

Privacy enforcement options:
(a) **Default privacy** — Each member sees only their own data. Admin can override per D-05.19 with audit. Requires extra work but models the shipped product correctly.
(b) **Admin transparency** — Admin (Piyush) sees all by default, audit trail only. Simpler to build, but doesn't test the privacy model that paying customers will rely on.

For trust, (a) is better. Beta testers need to see that hOS's privacy story is real, not just a future plan. Metin/Jes need to trust that their family's data is isolated from Piyush's admin access.

**Answer:** **(b) Admin transparency — simple for beta** — Admin (Piyush) sees all member data by default, audit trail only. Simpler to build, saves time for 4-person beta where trust is high. Privacy enforcement (per-member privacy with admin override) is designed but disabled for beta. Implement admin override flow (notice/delay/audit) in v2 when multi-household deployment is planned and strangers need privacy guarantees. Decided Aug 16, 2026.


### E4. Offline behavior — what works when the internet is down?

**The gap:** Vision success criterion #4: "Everything runs with the internet down except
external-service calls — and those queue and resume." But the model router needs an LLM,
and if the local model is down AND internet is down, the agent can't think.

**Decision needed:** What's the beta offline story? If internet is down, does the agent
stop entirely, fall back to on-device (Apple Foundation Models), or queue everything?
What do testers experience when offline?

**Analysis (prepared by requirements agent):**

From agent-context.md and v0.5.0 findings:
- Local LLM inference: Ollama (v0.32.9) or managed Ollama (D1 & D2) runs on reference machine
- Apple Foundation Models: Available (127.0.0.1:8003), on-device, no internet needed
- External services: Mail.app, Calendar.app (data cached locally), SimpleFIN (fetches cached transactions)
- Agent loop: Needs an LLM to synthesize, triage, draft

Options:
(a) **Online-first**: Agent requires internet/managed Ollama. Offline: user can read cached data but no synthesis. Simple to build, less impressive offline.
(b) **Hybrid with degradation**: Agent uses managed Ollama when available, falls back to Apple Foundation Models when offline. Slower but works. More complex.
(c) **Foundation Models only for beta**: All synthesis uses Apple Foundation Models (no Ollama dependency). Always works offline. Simplest, but limits model selection to Apple's choices.

Given D1 (Shared Capabilities first) which includes managed Ollama, and the goal of "everything works offline except external calls," option (b) is most aligned with vision. But (c) is simpler for beta and still proves the concept.

**Answer:** **(b) Hybrid with graceful degradation** — hOS prefers managed Ollama for synthesis when available. If internet is down or Ollama is unavailable, agent falls back to Apple Foundation Models for reads/synthesis/triage (slower, less capable, but works). External service calls (fetch new mail, transactions, calendar updates) queue and resume when internet returns. User sees: "Agent is working offline, responses will be slower" notification. Works for 4-person beta test, learns about model fallback before scaling to production. Decided Aug 16, 2026.


---

## F. The "Missing From 0.9 Scope" Items — features in Asana but not in the build plan

These are tagged for beta or in domain sections but have no corresponding 0.9 scope item:

### F1. Open Loops / Unified Queue
In Asana (beta-tagged, due Aug 21). NOT in 0.9 scope. Vision calls it the core primitive.
**Decision:** Build for 0.9 or defer? (See A2)

### F2. Whitelist / Autonomy Growth
In Asana (beta-tagged). NOT in 0.9 scope. Vision principle #1. The approval flow exists
but the whitelist (granting recurring autonomous actions) does not.
**Decision needed:** Is the whitelist mechanism needed for beta, or do all actions
require explicit approval in beta?

**Analysis (prepared by requirements agent):**

From A5 (Personal Agent Directives): Members define rules (whitelist = special rules allowing recurring autonomous action). For beta, the requirement is "member-created, member-managed rules" but the decision is whether they're enabled (autonomous action) or just recorded (still require approval).

Given C4 (agent learns patterns over time, automation graduates from behavior), and E1 (beta testers expect to help shape the agent), the whitelist MECHANISM should exist but be disabled for beta. Members can create rules, the system logs them, but agent still requires approval for actions. This lets beta testers learn how to specify rules without risky autonomous actions on day 1.

**Answer:** **(b) Full autonomy with whitelisting enabled** — Personal Agent Directives (Me tab) include member-created rules that govern brief timing, update frequency, whitelist rules, learning preferences. Rules are enabled from day 1 — when a rule is created and enabled, agent acts on it without approval. Testers learn autonomy boundaries by setting rules and seeing what the agent does. High-trust model but matches vision principle #1 (autonomy growth). Decided Aug 16, 2026.


### F3. Per-member Delegation
In Asana (not beta-tagged). NOT in 0.9 scope. Vision says household intelligence emerges
from coordination between member agents.
**Decision needed:** Is delegation (assign loops/tasks to another member's agent) needed
for beta or post-beta?

**Analysis (prepared by requirements agent):**

Delegation = one member's agent can create loops assigned to another member's attention/approval. Examples: Parent's agent creates "chore: take out trash" assigned to kid member. Spouse's agent creates "pay electric bill" assigned to primary household manager.

This is a household coordination feature, not core to A1-C5 (Mail vertical, Open Loops, iOS app, agent triage, Finance). It requires:
- Inter-agent communication protocol
- Assignment/notification system
- Per-member permission model (can A delegate to B?)

For 4-person beta (Piyush + Metin/Jes + Sophie), delegation would be nice (parents assign chores to Sophie), but it's not blocking. The core loop system works for per-member actions first.

**Answer:** **(c) Partial delegation — parents can assign to kids** — Household members with parent role (Metin, Jes) can create loops assigned to child members (Sophie) without their approval (chores, tasks). Other members (Sophie → Metin, Piyush → anyone) cannot assign. This supports the family chore use case without building full inter-member delegation protocol. General multi-member assignment deferred to v2. Decided Aug 16, 2026.


### F4. Memory v2 (pgvector)
In Asana (not beta-tagged). NOT in 0.9 scope (0.9 has Memory v1 SQLite only).
**Decision:** (See B2)

### F5. Process Isolation (XPC)
In Asana (not beta-tagged). NOT in 0.9 scope. LEARNINGS doc flags it.
**Decision:** (See B5)

### F6. Contract Versioning
In Asana (beta-tagged). NOT explicitly in 0.9 scope (but related to skill standard
alignment, item #5).
**Decision:** (See B6)

### F7. Cross-household / Second Install
In Asana (beta-tagged). NOT in 0.9 scope. D-05.16 names Metin + Jes as the test
household.
**Decision needed:** Is a second-install test part of beta, or is that a post-beta
activity? If beta, the installer must work on non-Piyush hardware.

**Analysis (prepared by requirements agent):**

D-05.16 (Households and Members) names: "Household 1 (Piyush + family on reference Mac mini, one install). Household 2 (Metin + Jes, separate install, stranger test)."

This is CRITICAL for beta validation. A6 decided self-service installer is a beta requirement. The installer must work for Metin/Jes without Piyush present. That's the second install test. It's not a "cross-household" feature (which would be multiple households syncing), it's proving the installer works for someone other than the dev.

For beta, both households run independently with no sync. This is the real test of "fresh install on clean Mac in under an hour."

**Answer:** **(a) Second install is part of beta** — Household 2 (Metin + Jes) gets a separate Mac mini running hOS. They use the self-service installer (A6) without Piyush present. Proves the installer works for non-dev users. No sync between Household 1 and 2 (that's v2). Single-household per install for beta. Household 2's data and loops are independent. Decided Aug 16, 2026.


### F8. Cross-domain Summarization (ELEVATION)
In Asana (not beta-tagged). NOT in 0.9 scope. Created from coverage audit.
**Decision needed:** Is summarization a shared capability (part of B1) or built
per-skill for beta?

**Analysis (prepared by requirements agent):**

From coverage audit: "Mail, Finance, Calendar, Notes, Messages, Journal — all need summarization." B1 (Shared Capabilities Architecture) includes summarization as a platform capability. All skills call `context.summarize(content:)` rather than implementing their own.

This is not a new feature — it's the design choice for B1. D1 decided "Shared Capabilities first, before any content skill." So summarization IS part of that architecture, not a separate decision.

**Answer:** **(a) Summarization as shared capability (confirms B1)** — Cross-domain summarization is part of the Shared Capabilities platform (B1), not per-skill. All content-processing skills (Mail, Finance, Journal, Calendar, Notes, Messages) call `context.summarize(content:)`. Implementation: managed Ollama + LLM (model selected via Model Router). First shared capability to build alongside Managed Ollama. Decided Aug 16, 2026.


---

## G. Research Topics Needing Direction

39 research topics exist. Most need more research before moving to Ready to Plan. But
some need direction now:

### G1. Shared Capabilities Architecture — move to Ready to Plan now?
This is the highest-priority research item. It blocks D1 and B1. If you agree it should
move to Ready to Plan, the coordination session can start planning it immediately.
**Decision:** Move to Ready to Plan now, or more research needed?

**Answer:** **(a) Move to Ready to Plan immediately** — B1 and D1 decisions confirm: Shared Capabilities Architecture (summarization/extraction/classification as platform APIs) is the highest-priority beta item. Ready-to-Plan trigger is met: vision documented (B1), competitive analysis done (Khoj, mem0), design direction clear (Managed Ollama + model manifest + subSkillCalls protocol). Move to Ready to Plan section now. Spec writing can start (docs/scope/shared-capabilities.md, feature/shared-capabilities branch). Decided Aug 16, 2026.


### G2. iPad Ambient Family Surface — research or beta?
The mockups exist. The research task has 7 subtask-areas already considered. Is this
ready to move to Ready to Plan, or does it need to stay in research until the privacy
scoping (public vs private) is resolved?
**Decision:** (Also relates to C2)

**Answer:** **(b) Move to Ready to Plan for beta** — C2 decided minimal shared surface for beta (Today view on iPad + household calendar + open loops). iPad Ambient Family Surface task moves to Ready to Plan now. Spec: adapt iOS app's Today tab layout to iPad screen size. No full wall-mounted display (that's v2 after user feedback), just the app running on shared iPad on the counter. Low effort, high visibility for families. Spec writing can start (docs/scope/ipad-today-view.md, feature/ipad-today-view branch). Decided Aug 16, 2026.


### G3. Podcast/Newsletter Knowledge Pipeline — defer until Shared Capabilities?
This depends on the Universal Knowledge Trigger and shared summarization/extraction
capabilities. Should it stay in research until those are planned?
**Decision:**

**Answer:** **(a) Defer to post-beta, keep in research** — Podcast/Newsletter pipeline requires: Universal KB Trigger (B3, not beta per decided G4), shared extraction capabilities (part of B1/Shared Capabilities), and semantic search (pgvector or NLEmbedding, B2 chosen). B1 and KB Trigger are both beta items NOW, but Podcast/Newsletter is enhancement on top. The pipeline is valuable research but not blocking beta. Keep task in Research Topics. Spec and build it in v2 after Shared Capabilities + KB Trigger + memory search are proven on Mail + Finance. Decided Aug 16, 2026.


### G4. Universal Knowledge Trigger — research or defer to post-beta?
This is an ambitious cross-cutting pattern. Is it needed for beta, or should all
knowledge features wait until post-beta?
**Decision:** (Also relates to B3)

**Answer:** **(a) Universal KB Trigger as platform pattern — build for beta** — B3 decision confirms: KB Trigger (all skills emit knowledge entities from day 1) is a beta platform pattern. It's not a user-facing feature — it's the architecture that makes semantic recall possible. Every skill built for beta (Mail, Finance, Calendar, Notes) will have the KnowledgeEmission protocol. MemoryStore stores extracted entities with NLEmbedding vectors. Podcast/Newsletter pipeline deferred (G3) because it's a CONSUMER of the KB trigger, not a part of its design. Foundation is set for beta. Decided Aug 16, 2026.


### G5. Product review backlog (21 products) — any that change the beta scope?
21 products were reviewed. The coverage audit identified 5 true gaps and 7 elevation
gaps. Most are post-beta research. But some (chores, meal planning, shared display) could
be beta differentiators.
**Decision:** Which product-inspired features, if any, should be pulled into beta scope?

**Answer:** **(a) Include the decided features, keep others as elevation for post-beta** — The 21-product review identified these as beta-relevant: chores (C3), meal planning (C5), shared display (C2), agent-layer email triage (C4), personal directives (A5). These are all decided and INCLUDED in beta. Other findings (Whimsical wall display from Cozi, rich recipe database from Mealime, delegation/collaboration from Todoist) are valuable future enhancements but not blocking beta scope. Elevation backlog: 7 post-beta features identified from audit, kept in Research Topics for v2. Coverage is complete for A1-C5 decisions. Decided Aug 16, 2026.


---

## Summary: The 5 Most Urgent Decisions

If you answer only 5 things before anything else:

1. **A1** — What IS the beta? (one vertical, both, or platform only)
2. **A2** — Is Open Loops in the beta? (the core primitive has no spec)
3. **B1** — When do we design Shared Capabilities? (blocks all content-processing skills)
4. **B2** — SQLite or pgvector for beta memory? (blocks knowledge features)
5. **D3** — Which of the 32 beta-tagged tasks are real beta blockers? (reconcile scope)

Everything else can follow, but these five determine the build plan shape.
