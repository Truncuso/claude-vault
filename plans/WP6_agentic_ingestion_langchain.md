# WP6: Agentic Ingestion — LangChain/LlamaIndex Migration

**Status**: specified
**Severity**: HIGH
**Created**: 2026-05-13
**Implemented**: —
**Depends on**: WP2, WP3

---

## Problem

The draft `agentic_note_ingestion` project at `Obsidian_Work_Vault/System/000_Scripts/agentic_note_ingestion/` uses smolagents for multi-agent orchestration but the user wants LangChain/LlamaIndex. It's missing: model discovery/selection, a prompt registry (prompts are embedded in Templater JS, not versioned), a podcast pipeline, an output quality evaluator, and a unified CLI entrypoint. The architecture is strong (JSON contracts, verification matrix, clean package structure) but the framework needs to change.

---

## Verified Evidence

- `ARCHITECTURE.md` — formal spec with optimization objective `J = alpha*R + beta*Q + gamma*T - delta*F`, smolagents-based
- `src/ingestion/youtube/orchestrator.py` — smolagents multi-agent orchestration
- `src/ingestion/pdf/orchestrator.py` — smolagents multi-agent orchestration
- `src/ingestion/common/config.py` — RuntimeConfig dataclass, env var resolution (can be reused)
- No `model_discovery.py`, `prompt_registry.py`, `evaluator.py`, `source_adapter.py` exist
- No `src/ingestion/podcast/` directory
- No `src/ingestion/cli/` directory
- No `prompts/` directory (prompts live in Templater JS templates in `System/00_Templates/`)
- Existing tests: `test_pdf_services_and_smoke.py`, `test_youtube_orchestrator_smoke.py`, `test_youtube_services.py`

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1 | Create `src/ingestion/common/model_discovery.py` | pending |
| 2 | Create `src/ingestion/common/prompt_registry.py` | pending |
| 3 | Create `src/ingestion/common/source_adapter.py` | pending |
| 4 | Create `src/ingestion/common/evaluator.py` | pending |
| 5 | Create `src/ingestion/podcast/` pipeline | pending |
| 6 | Refactor `src/ingestion/youtube/` smolagents → LangChain | pending |
| 7 | Refactor `src/ingestion/pdf/` smolagents → LangChain | pending |
| 8 | Create `src/ingestion/cli/main.py` unified entrypoint | pending |
| 9 | Create `prompts/` directory with versioned templates | pending |
| 10 | Update tests for LangChain migration | pending |
| 11 | Update ARCHITECTURE.md and SETUP.md | pending |

---

## Step-by-step

### Step 1 — Model Discovery

```python
@dataclass
class DiscoveredModel:
    name: str
    provider: Literal["ollama", "openai", "anthropic", "groq"]
    context_length: int
    available: bool
    capabilities: list[str]

def discover_models() -> list[DiscoveredModel]:
    """Query Ollama /api/tags, check env for API keys, return ranked models."""
    models = []
    # Ollama: GET http://$OLLAMA_BASE_URL/api/tags
    # OpenAI: check OPENAI_API_KEY env, query /models
    # Anthropic: check ANTHROPIC_API_KEY env
    return sorted(models, key=lambda m: m.context_length, reverse=True)

def recommend_model(task: str, constraints: ModelConstraints) -> DiscoveredModel:
    """Given a task type (summarization, extraction, embedding) and constraints,
    return the best available model."""
```

### Step 2 — Prompt Registry

```python
@dataclass
class PromptTemplate:
    name: str
    version: str
    model_constraints: ModelConstraints
    template: str  # Jinja2 template string

def load_prompt(name: str, version: str = "latest") -> PromptTemplate:
    """Load from prompts/<name>.<version>.yaml"""

def list_prompts() -> list[str]: ...
```

Prompts stored as YAML:
```yaml
# prompts/youtube_summary.v1.yaml
name: youtube_summary
version: "1.0"
model_constraints:
  min_context: 8192
  recommended_provider: ollama
  capabilities: [text]
template: |
  You are analyzing a YouTube video transcript.
  Title: {{ title }}
  Channel: {{ channel }}
  
  ## Transcript
  {{ transcript }}
  
  Produce a structured note with:
  1. Key takeaways (3-5 bullets)
  2. Detailed summary by topic sections
  3. Actionable insights
  4. Related concepts for wiki-linking
  
  Output as Markdown with YAML frontmatter (title, date, source_url, tags).
```

### Step 3 — Source Adapter Protocol

```python
class ContentSource(Protocol):
    source_type: str

    def extract(self, ref: str) -> ExtractionPayload: ...
    def chunk(self, payload: ExtractionPayload) -> list[Chunk]: ...
    def get_metadata(self, ref: str) -> dict: ...

class YouTubeSource(ContentSource):
    source_type = "youtube"
    # Uses yt-dlp for extraction, existing services.py for metadata

class PDFSource(ContentSource):
    source_type = "pdf"
    # Uses existing pdf/services.py extraction, adds LangChain chunking

class PodcastSource(ContentSource):
    source_type = "podcast"
    # NEW: whisper.cpp or OpenAI Whisper for transcription
```

### Step 4 — Evaluator

```python
@dataclass
class EvaluationReport:
    structure_compliance: float   # 0-1, does output match expected structure?
    completeness: float           # 0-1, are all requested sections present?
    hallucination_risk: str       # "low" | "medium" | "high"
    overall: str                  # "pass" | "fail" | "needs_review"
    issues: list[str]

def evaluate_output(payload: dict, criteria: list[str]) -> EvaluationReport:
    """Structural checks + optional LLM-based quality review."""
```

**Disclaimer:** `hallucination_risk` is a heuristic based on structural consistency checks and optional LLM review. LLM-based hallucination detection is circular (the evaluator LLM has the same hallucination tendency as the generating LLM). Treat as a warning flag, not a reliability guarantee. Future improvement: citation grounding (verify claims against source text), NLI-based entailment metrics, deterministic entity extraction checks.

### Step 5 — Podcast Pipeline

New pipeline mirroring the YouTube/PDF structure:
- `orchestrator.py` — LangChain-based orchestration
- `services.py` — Audio extraction, transcription (whisper), chunking
- Uses `source_adapter.PodcastSource` for unified interface

### Step 6-7 — LangChain Refactoring

Key changes from smolagents:
- Replace smolagents `Tool` classes with LangChain `@tool` decorators
- Replace smolagents agent hierarchy with LangChain `RunnableSequence` chains
- Use LangChain's `ChatOllama` / `ChatOpenAI` for model access
- Use LangChain's `StructuredOutputParser` for JSON contract compliance
- Keep existing extraction logic (services.py) — only replace orchestration

### Step 8 — Unified CLI

```bash
python -m ingestion.cli.main --source youtube --ref "https://..." --evaluate

# Input contract (JSON via stdin or flags)
{
  "source": {"type": "youtube", "ref": "https://..."},
  "model": {"provider": "ollama", "name": "auto"},
  "output": {"vault": "${OBSIDIAN_ACTIVE_VAULT}", "template": "YT_Local_Ingestion_v2"},
  "options": {"concurrency": 2, "evaluate": true}
}

# Output contract (stdout JSON)
{
  "status": "success",
  "model_used": {"provider": "ollama", "name": "glm-5:cloud"},
  "artifacts": [{"type": "note", "path": "..."}],
  "evaluation": {"structure_compliance": 0.95, "completeness": 0.87, "overall": "pass"},
  "timing": {"extraction_ms": 3200, "generation_ms": 12400, "total_ms": 15600}
}
```

### Step 9 — Prompt templates

```
prompts/
  youtube_summary.v1.yaml
  pdf_extraction.v1.yaml
  podcast_notes.v1.yaml
  transcript_sanitize.v1.yaml
```

### Step 10 — Updated tests

- `test_model_discovery.py` — mock Ollama API, verify model enumeration
- `test_prompt_registry.py` — load prompts, verify template rendering
- `test_evaluator.py` — known good/bad outputs, verify scores
- `test_source_adapter.py` — verify protocol conformance
- `test_podcast_pipeline.py` — mock audio, verify transcription → note flow
- Update `test_youtube_orchestrator_smoke.py` — LangChain pipeline
- Update `test_pdf_services_and_smoke.py` — LangChain pipeline

### Step 11 — Documentation

- Update `ARCHITECTURE.md`: new component diagram, LangChain patterns, evaluation framework
- Update `SETUP.md`: LangChain deps, whisper deps, model requirements

### Backward compatibility

**Step 0 (before any refactor): Audit existing entrypoints.**

Read both Templater templates and extract the exact `exec`/`execSync` calls. Document:
- Which Python module/function each template calls
- The expected input arguments (flags, env vars, stdin)
- The expected output format the template parses

Then preserve these entrypoints as thin wrappers that delegate to the new LangChain pipelines. Example:

```python
# Old entrypoint (kept for backward compat)
# youtube/orchestrator.py — preserve the old function signature
def main_legacy(url: str, output_dir: str) -> dict:
    """Thin wrapper — delegates to new LangChain pipeline."""
    from ingestion.cli.main import run_ingestion
    return run_ingestion(source="youtube", ref=url, output_dir=output_dir)
```

After migration, verify each Templater template executes successfully end-to-end. This is covered by test V5.4.

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| Python syntax | `python -m compileall src/ingestion/` | No errors |
| Import check | `python -c "import ingestion"` | No import errors |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V5.1 | Ingestion pipelines produce valid JSON contracts | Schema validation passes | JSON Schema validation on CLI output |
| V5.2 | Model discovery returns available models | count > 0, valid fields | `discover_models()`, verify structure |
| V5.3 | Evaluator produces numeric scores | All scores 0-1, overall is valid enum | Test with known good/bad outputs |
| V5.4 | Backward compat: old Templater templates work | Existing templates execute successfully | Run `YT_Local_Ingestion_v2.md` from Obsidian |
| T1 | YouTube ingestion produces structured note | Key takeaways, summary, insights sections present | Full pipeline with test video |
| T2 | PDF ingestion preserves section boundaries | Markdown headings match PDF structure | Full pipeline with test PDF |
| T3 | Podcast pipeline transcribes and structures | Transcript + structured note produced | Full pipeline with test audio |
| T4 | Model auto-selection picks best available | Falls back gracefully if preferred model unavailable | Kill Ollama, run ingestion |
| T5 | Prompt registry loads and renders templates | Jinja2 variables resolved correctly | Unit test with mock variables |
| T6 | Source adapter routes to correct pipeline | YouTube URL → YouTubeSource, PDF → PDFSource | Unit test with known refs |
| T7 | CLI handles errors with JSON contracts | Error output has error_code, message, details | Run with invalid URL |
| T8 | LangChain pipeline matches smolagents output quality | Comparable or better structure compliance | A/B comparison on test inputs |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| Whisper model download size (~1.5GB for medium) | MEDIUM | Document in SETUP.md |
| LangChain dependency version pinning | LOW | Use requirements.txt with frozen versions |
