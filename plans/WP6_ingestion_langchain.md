# WP6: Agentic Ingestion — LangChain/LlamaIndex Migration

**Status**: pending
**Severity**: HIGH
**Created**: 2026-05-14
**Depends on**: WP3, WP5

---

## Problem

The draft `agentic_note_ingestion` project at `Obsidian_Work_Vault/System/000_Scripts/agentic_note_ingestion/` uses smolagents for multi-agent orchestration but needs LangChain/LlamaIndex. Missing: model discovery/selection, a prompt registry, an output quality evaluator, a podcast pipeline, and a unified CLI entrypoint. Additionally, the ingestion pipeline must be style-aware — output format should match vault type and user's chosen style profile.

**Source:** `Obsidian_Work_Vault/System/000_Scripts/agentic_note_ingestion/`

---

## Implementation Steps

| Step | Deliverable | State |
|------|-------------|-------|
| 0 | **Audit existing entrypoints** — extract exact exec calls from Templater templates | pending |
| 1 | Create `src/ingestion/common/model_discovery.py` — Ollama + API model enumeration | pending |
| 2 | Create `src/ingestion/common/prompt_registry.py` — versioned prompts from YAML | pending |
| 3 | Create `src/ingestion/common/source_adapter.py` — unified ContentSource protocol | pending |
| 4 | Create `src/ingestion/common/evaluator.py` — output quality scoring | pending |
| 5 | Create `src/ingestion/podcast/` pipeline — audio → transcript → note | pending |
| 6 | Refactor `src/ingestion/youtube/` orchestrator smolagents → LangChain | pending |
| 7 | Refactor `src/ingestion/pdf/` orchestrator smolagents → LangChain | pending |
| 8 | Create `src/ingestion/cli/main.py` — unified CLI entrypoint | pending |
| 9 | Create `prompts/` directory with versioned templates | pending |
| 10 | **Style integration** — wire style profiles into output generation | pending |
| 11 | **Source auto-discovery** — script mode scanning vault for unprocessed URLs | pending |
| 12 | Update tests — model discovery, prompt registry, evaluator, podcast, LangChain pipelines | pending |
| 13 | Update ARCHITECTURE.md and SETUP.md | pending |

---

## Step 0 — Backward Compatibility Audit

Read both Templater templates and extract the exact `exec`/`execSync` calls:
- `YT_Local_Ingestion_v2.md` — which Python module/function, input args, expected output format
- `PDF_Agent_Ingestion.md` — which Python module/function, input args, expected output format

Preserve these as thin wrappers delegating to new LangChain pipelines.

---

## Step 1-4 — New Common Modules

**model_discovery.py:**
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

def recommend_model(task: str, constraints: ModelConstraints) -> DiscoveredModel:
    """Given task type and constraints, return best available model."""
```

**prompt_registry.py:**
```python
@dataclass
class PromptTemplate:
    name: str
    version: str
    model_constraints: ModelConstraints
    template: str  # Jinja2

def load_prompt(name: str, version: str = "latest") -> PromptTemplate: ...
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
template: |
  You are analyzing a YouTube video transcript.
  Title: {{ title }}
  ...
```

**source_adapter.py:**
```python
class ContentSource(Protocol):
    source_type: str
    def extract(self, ref: str) -> ExtractionPayload: ...
    def chunk(self, payload: ExtractionPayload) -> list[Chunk]: ...
    def get_metadata(self, ref: str) -> dict: ...

class YouTubeSource: ...
class PDFSource: ...
class PodcastSource: ...  # NEW: whisper transcription
```

**evaluator.py:**
```python
@dataclass
class EvaluationReport:
    structure_compliance: float   # 0-1
    completeness: float           # 0-1
    hallucination_risk: str       # "low" | "medium" | "high" (heuristic only)
    ai_pattern_score: float       # 0-1 (from WP5 anti-pattern filter)
    overall: str                  # "pass" | "fail" | "needs_review"
    issues: list[str]
```

**Disclaimer on hallucination_risk:** LLM-based hallucination detection is circular (evaluator LLM has same tendency as generating LLM). Treat as warning flag, not reliability guarantee. Future: citation grounding, NLI-based entailment, deterministic entity extraction checks.

---

## Step 6-7 — LangChain Refactoring

- Replace smolagents `Tool` classes with LangChain `@tool` decorators
- Replace agent hierarchy with LangChain `RunnableSequence` chains
- Use `ChatOllama` / `ChatOpenAI` for model access
- Use `StructuredOutputParser` for JSON contract compliance
- **Keep existing extraction logic (services.py)** — only replace orchestration

---

## Step 8 — Unified CLI

```bash
python -m ingestion.cli.main --source youtube --ref "https://..." --style professional --evaluate

# Input contract
{"source": {"type": "youtube", "ref": "https://..."}, "model": {"provider": "ollama", "name": "auto"}, "output": {"vault": ".", "style": "professional"}, "options": {"evaluate": true, "output_mode": "structured"}}

# Output contract (structured mode)
{"status": "success", "model_used": {"provider": "ollama", "name": "..."}, "artifacts": [{"type": "note", "path": "..."}, {"type": "transcript", "path": "..."}], "evaluation": {"structure_compliance": 0.95, "completeness": 0.87, "ai_pattern_score": 0.02, "overall": "pass"}, "style_applied": "professional", "timing": {"extraction_ms": 3200, "generation_ms": 12400, "total_ms": 15600}, "output_mode": "structured"}
```

**Output modes:**

| Mode | Flag | Behavior | Use case |
|------|------|----------|----------|
| `structured` (default) | `--output structured` | Full pipeline: extract → summarize → style-transform → anti-pattern-filter → structured note | Normal ingestion |
| `raw-transcript` | `--output raw-transcript` | Extract transcript/text only, save as raw note with source metadata. No LLM summarization. Useful for manual review, debugging extraction, or when LLM is unavailable. | Bypass LLM, pass-through only |

---

## Step 10 — Style Integration

Ingestion pipeline reads active style profile from WP5, transforms output accordingly:
```
Raw transcript → extract key content → prompt_registry template → LLM generation → style transform → anti-pattern filter → structured note
```

---

## Step 11 — Source Auto-Discovery

Script mode that scans vault for YouTube URLs, podcasts links, unprocessed PDFs:
```bash
python -m ingestion.cli.main --mode scan --vault . --since 7d
```

Finds unprocessed URLs → extracts transcripts → generates style-formatted notes → files in appropriate vault folder.

---

## Verification

| ID | Test | Expected | Method |
|----|------|----------|--------|
| V6.1 | Pipelines produce valid JSON contracts | Schema validation passes | JSON Schema validation |
| V6.2 | Model discovery returns available models | count > 0, valid fields | `discover_models()` |
| V6.3 | Evaluator produces numeric scores | All 0-1, overall valid enum | Test with fixture outputs |
| V6.4 | Backward compat: old templates work | Existing Templater templates execute | Run `YT_Local_Ingestion_v2.md` and `PDF_Agent_Ingestion.md` |
| T6.1 | YouTube pipeline produces structured note | Key takeaways, summary, insights | Full pipeline with test video |
| T6.2 | PDF pipeline preserves section boundaries | Markdown headings match PDF structure | Test with multi-section PDF |
| T6.3 | Podcast pipeline transcribes and structures | Transcript + structured note | Test with sample audio |
| T6.4 | Model auto-select falls back gracefully | Uses next-best if preferred unavailable | Kill Ollama, run ingestion |
| T6.5 | Prompt registry loads and renders templates | Jinja2 vars resolved | Unit test with fixture data |
| T6.6 | Source adapter routes by ref type | youtube.com → YouTubeSource, .pdf → PDFSource | Unit test with various refs |
| T6.7 | CLI error handling via JSON contract | error_code, message, details | Run with invalid/missing source |
| T6.8 | Timing info in CLI output | extraction_ms, generation_ms, total_ms > 0 | Parse output, verify fields |
| T6.9 | Style-aware output respects profile | Output matches target style | Compare casual vs academic output from same source |
| T6.10 | Anti-pattern filter runs on ingestion output | AI-isms flagged in evaluation | Ingest known AI-generated content |
