# WP8: Learning Vault

**Status**: draft (specified for later)
**Severity**: LOW
**Created**: 2026-05-14
**Depends on**: WP3, WP5

**Skill inheritance:** Learning vault includes ALL WP3 general skills plus the learning-tier skills below.

---

## Problem

Structured learning from courses, books, and tutorials needs dedicated skills: course progress tracking, book note generation, concept extraction and mapping, and spaced repetition scheduling. These differ from working vault skills in their focus on structured curriculum tracking and knowledge retention.

---

## Planned Skills

| Skill | Purpose |
|-------|---------|
| `course-tracker` | Module/chapter progress, deadlines, syllabus parsing |
| `book-ingest` | Structured book note generation from chapters/sections |
| `concept-mapper` | Concept extraction, wikilink graphing, knowledge graph generation |
| `spaced-repetition` | Review scheduling based on note metadata, spaced repetition prompts |
| `exercise-solver` | Problem set assistance with step-by-step reasoning (optional) |

---

## Planned Templates

- Course Overview Template.md
- Lecture Note Template.md
- Book Chapter Template.md
- Concept Page Template.md
- Exercise Log Template.md

---

## Open Questions

1. **Course syllabus parsing:** Auto-parse from PDF/web or manual entry?
2. **Spaced repetition integration:** Use Anki/AnkiConnect or custom note metadata + QMD queries?
3. **Concept mapping:** How to auto-extract concepts and link them? Use LLM extraction + wikilink generation?
4. **Book ingest:** Chapter-by-chapter or batch? Audio book support (podcast pipeline reuse)?

---

## Architecture Notes

- `concept-mapper` reuses patterns from ARIS `research-wiki` (entity extraction, typed edges)
- `book-ingest` reuses WP6 ingestion pipeline (PDF → structured note, style-aware)
- `spaced-repetition` uses QMD to query notes by review date, frontmatter-based scheduling

---

## Verification (When Implemented)

- [ ] `course-tracker` parses syllabus and creates module pages
- [ ] `book-ingest` generates chapter notes with concept wikilinks
- [ ] `concept-mapper` builds knowledge graph from existing notes
- [ ] `spaced-repetition` schedules reviews based on note metadata
