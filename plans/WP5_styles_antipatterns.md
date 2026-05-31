# WP5: Configurable Note Styles & Anti-Pattern Filter

**Status**: pending
**Severity**: MEDIUM
**Created**: 2026-05-14
**Depends on**: WP3

---

## Problem

Note generation quality depends on vault context. A daily journal entry should sound casual; a project brief should be professional; a research paper note should be academic. Currently, all notes use the same generic AI voice. Additionally, LLM-generated notes often contain AI-writing patterns (filler phrases, chatbot artifacts, significance inflation) that pollute a personal knowledge base.

**Solution:** Configurable style profiles per vault type + `avoid-ai-writing` integration as anti-pattern filter.

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1 | Create `prompts/styles/` — YAML style profiles | pending |
| 2 | Create style detection logic — vault-type → default profile | pending |
| 3 | Integrate `avoid-ai-writing` 36 patterns as post-processing filter | pending |
| 4 | Wire style system into note-creation and ingestion pipelines | pending |

---

## Style Profile Format

```yaml
# prompts/styles/technical.yaml
name: technical
voice: precise, declarative
sentence_length: medium
formality: high
pronouns: [it, they]        # Avoid I/we/you
features:
  code_blocks: true
  diagrams: true
  bullet_ratio: 0.4         # 40% bullet content
avoid:
  - filler_phrases
  - chatbot_artifacts
  - significance_inflation
  - rhetorical_questions
word_tiers:
  tier1_always_flag: [delve, leverage, tapestry, robust, ecosystem]
  tier2_cluster_flag: [harness, foster, realm, landscape]
```

## Style Profiles (Initial Set)

| Profile | Voice | Vault Types | Example Use |
|---------|-------|-------------|-------------|
| `casual` | Personal, short sentences, contractions | Working (daily notes) | Journal entries, quick captures |
| `professional` | Formal but concise, action-oriented | Working (meetings, projects) | Meeting notes, project briefs |
| `technical` | Code blocks, precise terminology | Working (dev notes) | Technical documentation |
| `academic` | Citations, formal structure, hedging | Research | Paper notes, literature reviews |
| `minimal` | Bullet-only, zero prose | All (reference) | Pure information density |

---

## Style Detection

```
Detection priority:
1. Explicit --style flag (user override)
2. Note frontmatter style: field
3. Note path pattern (/System/ → technical, /Daily/ → casual)
4. Vault type default (working → professional, research → academic)
5. Global fallback (professional)
```

## Anti-Pattern Integration

From `avoid-ai-writing` (v3.3.1, Conor Bronsdon):

**36 pattern categories across 5 groups integrated as post-processing filter:**
- Content: significance inflation, notability name-dropping, superficial -ing analyses
- Language: 109-entry word replacement table (3 tiers), copula avoidance, synonym cycling
- Structure: em dashes, uniform paragraph length, formulaic openings
- Communication: chatbot artifacts ("Certainly!", "I hope this helps!"), sycophantic tone
- Meta: rhythm/uniformity (#1 detection signal per Pangram Labs)

**Pipeline integration:**
```
Raw generation → Style transform → Anti-pattern scan → P0/P1/P2 flag → Apply fixes → Final note
```

P0 flags (credibility killers) block note creation. P1 flags (obvious AI smell) auto-tag notes for review. P2 (stylistic polish) informational only.

---

## Architecture Note

Style system is architecture-specified in WP5 but full implementation of all 5 profiles and all 36 pattern categories is deferred. The base architecture (profile files, detection hooks, filter pipeline) is built in WP5; per-vault-type profile tuning and full anti-pattern coverage is ongoing work.

---

## Verification

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V5.1 | All 5 style profiles parse as valid YAML | No parse errors | `python -c "import yaml; yaml.safe_load(...)"` |
| V5.2 | Style detection returns correct profile | Correct for vault type | Test with each vault type |
| T5.1 | Anti-pattern filter catches known AI-isms | Tier 1 words flagged | Test with AI-generated sample |
| T5.2 | Style transform applies target voice | Output matches profile | A/B: styled vs unstyled output |
| T5.3 | P0 flags block note creation | Note not created when credibility killer found | Test with P0-trigger content |
| T5.4 | Style override via --style flag works | Manual flag takes priority | Test --style flag |
| T5.5 | Style override via frontmatter works | Frontmatter style: field takes priority | Test with style: casual |
