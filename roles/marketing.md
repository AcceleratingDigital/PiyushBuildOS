# Marketing Site Agent — Learned Context

Read this file before updating the marketing site. It accumulates the design
system, page templates, and update procedures.

## Site structure

```
site/                          # in repo (source of truth)
  index.html                   # landing page
  system.html                  # how it works
  mockups.html                 # app preview
  design-system.html           # design tokens
  status.html                  # project status (auto-generated from Asana)
  process.html                 # how we build hOS (marketing, pipeline visual)
  releases/
    index.html                 # releases index
    v0-3-4.html                # per-release notes
  docs/
    index.html                 # docs landing
    skills/
      notes.html               # per-skill user docs
      journal.html
      ...
```

## CRITICAL: Edit in repo, then sync

1. **Edit files in `site/` in the repo** (`~/code/hos-monorepo/site/`)
2. **Commit to git**
3. **Sync to live:** `rsync -av --delete ~/code/hos-monorepo/site/ /Volumes/web/local/ad/hos/`
4. **NEVER edit `/Volumes/web/` directly** — it's a deployment target

## Design system

- CSS variables defined in `design-system.html` and each page's `<style>` block
- Light/dark mode via `prefers-color-scheme`
- Fonts: system UI, serif for lede text, mono for code
- Colors: ground (#EDE8DF light / #0F1216 dark), structure (#44607F), accent (#B4741E), ok (#1E6E5A)
- Badges: shipped (🟢), in progress (🟡), planned (⚪), 0.9 build (🚧)

## Status page (status.html)

- Auto-generates from Asana tags (not manually edited)
- Hermes runs a script that queries Asana, aggregates statuses, generates HTML
- 9 sections, 81 features, mini progress bars per feature
- Summary stats at top (released, QA passed, in progress, ready, research, blocked)
- **CRITICAL: Has a filter bar with clickable stage filters + JS.** When regenerating from Asana data, PRESERVE the filter CSS (`.filters`, `.filter-btn`, `.hidden`, `.no-results`), the filter bar HTML (`<div class="filters">`), the no-results div, and the `<script>` block. Only replace the data section (`.summary`, `.track-segments`, `.feature` cards). If you rebuild from scratch you WILL lose the filtering. Always start from the existing file and swap data only.

## Doc page template

Match `docs/skills/notes.html` structure:
- Same nav, footer, CSS
- Status badge (🚧 0.9 build or 🟢 shipped)
- Sections: What it does, How to use, Setup, Privacy, Troubleshooting, Security
- Written for non-technical household members
- No internal jargon, no architecture diagrams

## Update triggers

- New skill built → add doc page + update docs/index.html
- Skill shipped → update badge from 🚧 to 🟢
- Feature status changes → regenerate status.html
- Release published → update footer version, capability table

## Past work

| Date | Page | Change |
|---|---|---|
| 2026-08-15 | status.html | Created — 81 features, 9 sections |
| 2026-08-15 | docs/skills/notes.html | Created — Notes Read user docs |
| 2026-08-15 | docs/skills/journal.html | Created — Journal Read user docs |
| 2026-08-15 | index.html | Added Status nav link, capability table |
| 2026-08-15 | process.html | Created — pipeline visual, agent team, two review gates |
