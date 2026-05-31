# WP4: Working Vault Skills + Templater Templates

**Status**: pending
**Severity**: MEDIUM
**Created**: 2026-05-14
**Depends on**: WP3

---

## Problem

General skills (WP3) provide core operations but working vaults need specialized workflows: structured daily reflection, project status tracking, rapid capture, and meeting documentation. Templater templates must ship with the plugin so agents can generate high-quality, consistently-formatted notes.

---

## Implementation Steps

| Step | Deliverable | State |
|------|-------------|-------|
| 1 | `daily-journal` skill — structured daily reflection | pending |
| 2 | `project-tracker` skill — status extraction, next-action generation | pending |
| 3 | `quick-capture` skill — inbox-style rapid note creation | pending |
| 4 | `meeting-notes` skill — template-driven meeting documentation | pending |
| 5 | Templater templates (5 templates) | pending |
| 6 | Template installation via setup skill | pending |

---

## Skill Specifications

### 1. daily-journal

Structured end-of-day or morning reflection. Prompts guide the user through sections.

```yaml
name: daily-journal
description: "Structured daily reflection. Prompts for accomplishments, blockers, learnings, tomorrow's focus. Appends to or creates daily note."
argument-hint: "[--mode morning|evening|prompt]"
```

**Workflow:** Load daily note template → prompt user for each section → format responses → append/prepend to daily note → verify frontmatter.

### 2. project-tracker

Scans project directories, extracts status from frontmatter, generates next-action lists.

```yaml
name: project-tracker
description: "Track project status. Scan project folders, extract frontmatter status, generate next-action lists. Creates project dashboards."
argument-hint: "[project-name] [--action status|next|dashboard]"
```

### 3. quick-capture

Minimal-friction note creation for fleeting thoughts, ideas, or to-dos.

```yaml
name: quick-capture
description: "Rapid inbox-style note capture. Minimal friction. Auto-files to Inbox/ with minimal frontmatter. Processed later via project-tracker or manual review."
argument-hint: "[content] [--type idea|todo|note|bookmark]"
```

### 4. meeting-notes

Template-driven meeting documentation with participant tracking, decisions, and action items.

```yaml
name: meeting-notes
description: "Template-driven meeting documentation. Prompts for participants, agenda, decisions, action items. Generates structured meeting note with frontmatter."
argument-hint: "[meeting-title] [--template <name>]"
```

---

## Templater Templates

Shipped with the plugin, installed to vault's template folder by the setup skill.

### Daily Note Template.md

```markdown
---
date: <% tp.date.now("YYYY-MM-DD") %>
type: daily-note
tags: [daily]
---

# <% tp.date.now("dddd, MMMM D, YYYY") %>

## Focus for Today
- [ ] 

## Notes

## Accomplishments

## Learnings

## Tomorrow
- [ ] 
```

### Meeting Note Template.md

```markdown
---
date: <% tp.date.now("YYYY-MM-DD") %>
type: meeting
participants: []
tags: [meeting]
---

# <% tp.file.title %>

**Date:** <% tp.date.now("YYYY-MM-DD HH:mm") %>
**Participants:** 

## Agenda

## Notes

## Decisions

## Action Items
- [ ] 
```

### Project Brief Template.md

```markdown
---
status: active
priority: medium
start_date: <% tp.date.now("YYYY-MM-DD") %>
tags: [project]
---

# <% tp.file.title %>

## Objective

## Key Results

## Current Status

## Next Actions
- [ ] 

## Notes
```

### Quick Capture Template.md

```markdown
---
date: <% tp.date.now("YYYY-MM-DD HH:mm") %>
type: inbox
source: quick-capture
---

# <% tp.file.title %>

<% tp.file.cursor() %>
```

### Weekly Review Template.md

```markdown
---
date: <% tp.date.now("YYYY-MM-DD") %>
type: weekly-review
week: <% tp.date.now("gggg-[W]ww") %>
tags: [review, weekly]
---

# Weekly Review — Week <% tp.date.now("ww") %>

## Wins

## Challenges

## Projects Progress

## Learnings

## Next Week Focus
```

---

## Verification

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V4.1 | All 4 working-vault skills load | SKILL.md parsed, description displayed | Skill invocation |
| V4.2 | daily-journal produces structured entry | All sections filled, valid frontmatter | E2E with test prompts |
| T4.1 | project-tracker extracts project status | Correct status from frontmatter | Run on known project file |
| T4.2 | quick-capture creates inbox note | Note appears in Inbox/, minimal frontmatter | Capture test content |
| T4.3 | meeting-notes generates structured note | Participants, decisions, actions present | Generate from test meeting |
| T4.4 | All 5 templates installed correctly | Files exist in vault template folder | `test -f` each template |
| T4.5 | Templates produce valid markdown | Valid frontmatter, correct structure | Render each template |
| T4.6 | Notes created from templates appear in Obsidian | Visible in file explorer | Manual: check Obsidian UI |
