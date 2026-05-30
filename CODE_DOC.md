# Code Documentation — Synapt.AI Dataset Pipeline

Developer-level documentation covering the architecture, data flow, and implementation details of the synthetic dataset generation pipeline.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [How the LLM API Creates Synthetic Data](#how-the-llm-api-creates-synthetic-data)
3. [Module-by-Module Breakdown](#module-by-module-breakdown)
   - [config.py](#configpy)
   - [run_pipeline.py](#run_pipelinepy)
   - [pipeline/extract.py](#pipelineextractpy)
   - [pipeline/chunk.py](#pipelinechunkpy)
   - [pipeline/azure_client.py](#pipelineazure_clientpy)
   - [pipeline/prompts.py](#pipelinepromptspy)
   - [pipeline/generate.py](#pipelinegeneratepy)
   - [pipeline/quality.py](#pipelinequalitypy)
   - [pipeline/progress.py](#pipelineprogresspy)
   - [pipeline/split.py](#ipelinesplitpy)
4. [Data Flow & Intermediate Formats](#data-flow--intermediate-formats)
5. [Unsupervised Finetuning — Why This Approach](#unsupervised-finetuning--why-this-approach)
6. [Rate Limiting & Concurrency Model](#rate-limiting--concurrency-model)
7. [Checkpoint/Resume System](#checkpointresume-system)
8. [Extending the Pipeline](#extending-the-pipeline)

---

## Architecture Overview

The pipeline follows a five-phase linear architecture:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────────┐    ┌─────────────────┐    ┌──────────────────┐
│  EXTRACTION  │───>│   CHUNKING   │───>│    GENERATION     │───>│ QUALITY CONTROL │───>│  LINEAGE SPLIT   │
│              │    │              │    │   (15 strategies)  │    │                 │    │                  │
│  PDF -> Text │    │ Text -> JSONL│    │   LLM API calls   │    │  Merge + Dedup  │    │ Union-Find group │
│  + Sections  │    │  6 tiers     │    │   + lineage IDs   │    │  + Filter       │    │ -> train/val/test│
└──────────────┘    └──────────────┘    └──────────────────┘    └─────────────────┘    └──────────────────┘
```

Each phase writes its output to disk and marks completion in a state file. This makes the pipeline:
- **Resumable**: interrupted runs skip completed work on restart
- **Inspectable**: every intermediate artifact is a readable JSONL or JSON file
- **Idempotent**: re-running a completed phase is a no-op
- **Domain-agnostic**: domain is configured via `config.DOMAIN`, propagated to all prompts and quality filters automatically

The orchestrator (`run_pipeline.py`) is an `async def main()` function driven by `asyncio.run()`. Async is used exclusively in Phase 3 (generation) where concurrent API calls provide significant throughput gains. Phases 1, 2, 4, and 5 are synchronous.

---

## How the LLM API Creates Synthetic Data

This is the core insight of the pipeline. The goal is to produce a large, diverse text corpus from a small number of source PDFs, suitable for unsupervised finetuning (also called continued pretraining) of a language model. Here is how that works, step by step:

### The Problem

You have a few hundred pages of technical documentation. Directly finetuning on this raw text is ineffective because:
- The text is repetitive (headers, footers, boilerplate)
- It exists in only one "voice" and style
- Tables and structured data don't translate well to language model training
- There isn't enough volume — a few hundred pages produce thousands of tokens, but finetuning benefits from tens of thousands of diverse examples

### The Solution: Multi-Strategy LLM Augmentation

Instead of training on the raw text, the pipeline uses an LLM (Azure OpenAI GPT-4.1) to create **semantic variations** of the source material. Each chunk of source text is sent to the LLM with a carefully designed prompt that asks it to transform the content in a specific way. The pipeline defines 15 such transformation strategies:

1. **Style Variation** (`paraphrase`, `micro_paraphrase`, `paraphrase_extra`) — 3 strategies: The same content rewritten in 5–10 different styles — formal, conversational, concise, simplified, implementation-focused, training-oriented, executive summary, troubleshooting guide, academic, blog post. `paraphrase` runs on `standard`-tier chunks; `micro_paraphrase` produces 5 short rewrites for `micro`-tier chunks; `paraphrase_extra` re-applies the 10-style template to the `extra_overlap` tier (50% overlap variants) for additional volume. This teaches the finetuned model to handle the domain across registers.

2. **Audience Variation** (`concept_explanation`) — 1 strategy: The same concept explained at 5 expertise levels — beginner, intermediate, expert, architect, operator. Runs on `macro`-tier chunks for richer context. This gives the finetuned model depth of understanding.

3. **Format Transformation** (`definition`, `procedural`, `summary`, `qa_expository`, `table_narration`) — 5 strategies: Content reshaped into definitions, step-by-step procedures, summaries at different granularities, Q&A-style expositions, and natural-language descriptions of tabular data. This teaches the model to work with domain knowledge in different formats.

4. **Analytical Enrichment** (`cross_reference`, `expansion`, `contrastive`, `deep_dive`) — 4 strategies: The LLM is asked to analyze relationships between sections, expand on concepts, compare/contrast ideas, and provide deep architectural analysis. This creates training data that captures implicit knowledge not directly stated in the source.

5. **Contextual Application** (`scenario`, `terminology_context`) — 2 strategies: The LLM generates realistic operational scenarios and uses domain terms in realistic technical contexts. This grounds the domain knowledge in practical usage patterns.

Total: 3 + 1 + 5 + 4 + 2 = **15 strategies**.

### How a Single API Call Works

Taking `paraphrase` as an example:

1. A standard-tier chunk (256 tokens) is selected
2. The chunk text is injected into the `paraphrase` prompt template at the `{chunk_text}` placeholder
3. The prompt instructs the LLM to produce 10 rewrites, each preceded by a `[STYLE_N]` marker
4. A system prompt establishes the LLM's role as a domain expert (built dynamically from `config.DOMAIN`)
5. The API call returns a response containing all 10 rewrites separated by markers
6. The response is parsed by splitting on the `[STYLE_N]` markers using regex
7. Each parsed segment becomes an independent training example in the output JSONL

The key parameters:
- **Temperature 0.7**: Enough randomness for diversity, enough determinism for accuracy
- **Max tokens 4000**: Accommodates the largest strategy outputs (10 rewrites)
- **System prompt**: Anchors the LLM to the configured domain, preventing hallucination drift

### Why This Produces Good Training Data

- **Semantic preservation**: Every output is grounded in real source content — the LLM transforms but doesn't fabricate domain knowledge
- **Diversity**: 14 strategies × multiple outputs per call × 6 chunk tiers = massive combinatorial diversity from limited source material
- **Domain richness**: The system prompt and source grounding ensure domain-specific terminology and concepts are preserved
- **Lineage tracking**: Every generated row carries `parent_chunk_id` linking it to its source chunk, enabling leakage-free train/test splits
- **No labels needed**: The output is pure text, ready for unsupervised finetuning without annotation

---

## Module-by-Module Breakdown

### config.py

Central configuration module. All magic numbers and paths are defined here as module-level constants.

**Path Management**: Uses `pathlib.Path` to define `BASE_DIR`, `INPUT_DIR`, `OUTPUT_DIR`, and subdirectories (`EXTRACTED_DIR`, `CHUNKS_DIR`, `GENERATED_DIR`, `FINAL_DIR`, `PROGRESS_DIR`, `LOG_DIR`). `config.py` only declares paths — it does NOT create directories. Each consumer creates the directories it needs lazily via `mkdir(parents=True, exist_ok=True)` (e.g. `extract_all_pdfs`, `save_chunks`, `run_strategy`, `PipelineProgress.__init__`).

**Chunking Constants**:
```python
CHUNK_SIZES = {"micro": 128, "standard": 256, "section": 512, "macro": 1024}
PRIMARY_CHUNK_SIZE = 256
OVERLAP_RATIO = 0.2          # 20% token overlap between consecutive chunks
EXTRA_OVERLAP_RATIO = 0.5    # 50% overlap for the extra_overlap tier
TOKENIZER_ENCODING = "cl100k_base"  # GPT-3.5/4 tokenizer
```

**API Constants**: `MAX_CONCURRENT_REQUESTS` (5), `REQUESTS_PER_MINUTE` (30), `TOKENS_PER_MINUTE` (80,000), `GENERATION_TEMPERATURE` (0.7), `GENERATION_MAX_TOKENS` (4000).

**Quality Constants**: `FUZZY_DEDUP_THRESHOLD` (0.85), `MINHASH_NUM_PERM` (128), `MIN_CHUNK_TOKENS` (30), `MAX_CHUNK_TOKENS` (2048), `MAX_SPECIAL_CHAR_RATIO` (0.4).

**Pipeline Constants**: `TARGET_CHUNKS` (50,000), `CHECKPOINT_INTERVAL` (100).

**Domain Constants**: `DOMAIN` (`"telecommunications"` — change for other domains), `DOMAIN_DESCRIPTION` (optional system prompt specialization), `DOMAIN_SEED_TERMS` (optional seed terms for quality filtering; empty by default, auto-derived from PDFs). These three constants control all domain-specific behavior across prompts and quality filtering.

**Dataset Split Constants**:
```python
DATASET_SPLIT_ENABLED = True
DATASET_SPLIT_STRATEGY = "lineage"    # extensible: random, lineage, document, hierarchical, temporal
DATASET_SPLIT_RATIOS = {"train": 0.90, "validation": 0.05, "test": 0.05}
DATASET_SPLIT_SEED = 42              # for reproducible splits
DATASET_SPLIT_FREEZE_MAPPING = True  # persist lineage->split mapping across runs
DATASET_SPLIT_MAPPING_FILE = FINAL_DIR / "lineage_map.json"
```

**Schema Constants**: `DATASET_SCHEMA_VERSION` (`"2.0"`) — stamped into every output row.

---

### run_pipeline.py

The top-level orchestrator. Contains `async def main()` which runs the five phases sequentially.

**Phase 1 — Extraction**:
```python
if not progress.extraction_complete:
    documents = extract_all_pdfs(INPUT_DIR, EXTRACTED_DIR)
    progress.extraction_complete = True
```
Calls into `pipeline.extract`, stores the returned `Document` objects for Phase 2.

**Phase 2 — Chunking**:
```python
if not progress.chunking_complete:
    all_doc_chunks = {}
    for doc in documents:
        all_doc_chunks[doc.name] = chunk_document(doc)  # returns dict[tier, list[Chunk]]
    total_raw = save_chunks(all_doc_chunks, str(CHUNKS_DIR))  # writes JSONL files
    progress.chunking_complete = True
```
`chunk_document` returns a dict mapping tier name to chunks (it does NOT write files). The separate `save_chunks` function is responsible for writing per-document, per-tier JSONL files (`{pdf_name}_{tier}.jsonl`).

**Phase 3 — Generation**:
```python
chunks_by_tier = _load_chunks_from_dir(CHUNKS_DIR)
client = AzureGenerationClient()
total = await run_all_strategies(chunks_by_tier, client, progress, GENERATED_DIR)
await client.close()
```
Loads all chunks from disk grouped by tier, creates the Azure client, and delegates to the generation orchestrator. The client is closed after all strategies complete.

**Phase 4 — Quality Control**:
```python
all_chunks = []
# Add raw chunks from each tier (tagged strategy="raw_extraction", with lineage)
for tier, chunks in chunks_by_tier.items():
    for chunk in chunks:
        all_chunks.append({
            "text": chunk.text,
            "source_pdf": chunk.source_pdf,
            "source_section": chunk.source_section,
            "strategy": "raw_extraction",
            "parent_chunk_id": chunk.id,
        })
# Append generated chunks (now include parent_chunk_id from Phase 3)
generated = _load_generated_chunks(GENERATED_DIR)
all_chunks.extend(generated)

raw_texts = [doc.raw_text for doc in documents]
domain_terms = extract_domain_terms(raw_texts)
final_chunks = run_quality_pipeline(all_chunks, domain_terms)
```
**Raw chunks are merged with generated chunks** before quality filtering — the final dataset includes both. Each raw chunk carries its own `chunk.id` as `parent_chunk_id` (a raw chunk is its own ancestor). Domain terms are auto-extracted from source documents for quality validation.

**Phase 5 — Lineage Split & Output**:
```python
enriched_rows = run_split_pipeline(final_chunks, str(config.FINAL_DIR))
_write_per_pdf_output(enriched_rows, str(config.FINAL_DIR), pdf_names)
```
After quality filtering, the split pipeline groups rows by lineage, assigns splits, enriches rows with schema v2.0 metadata, and writes all output files (train/val/test JSONL, combined dataset, manifest, lineage map). Per-PDF files are written separately for backward compatibility.

**Helper Functions**:
- `_load_chunks_from_dir(dir)`: Reads all `*.jsonl` files, groups chunks by their `tier` field into a `dict[str, list[Chunk]]`
- `_load_generated_chunks(dir)`: Reads all strategy JSONL files into a flat list (preserving all fields including `parent_chunk_id`)
- `_write_per_pdf_output(rows, output_dir, pdf_names)`: Writes per-PDF JSONL files using the `source_id` field from enriched v2.0 rows, with partial-match fallback for document name matching

---

### pipeline/extract.py

Handles PDF-to-text conversion with structure preservation.

**Data Classes**:

- `Table`: Represents a detected table with `page`, `rows` (raw cells), `markdown`, `narration` (natural language), and `csv_text` representations
- `Section`: A document section from the TOC — `title`, `level`, `page_start`, `page_end`, `text`, `tables`, `full_path` (e.g., "Chapter 3 > Billing > Rate Plans")
- `Document`: Top-level container with `name`, `pdf_path`, `total_pages`, `sections`, `flat_sections`, `raw_text`

**Text Cleaning** (`_clean_text`):
- Strips null bytes (`\x00`) and BOM markers (`\ufeff`)
- Normalizes whitespace: collapses runs of 3+ newlines to 2, strips trailing spaces
- Fixes hyphenation: joins words split across lines (e.g., `"bill-\ning"` -> `"billing"`)

**TOC Extraction** (`_extract_toc`):
- Uses PyMuPDF's `doc.get_toc()` which returns `[level, title, page]` tuples
- Falls back to an empty list (creating a single "Document" section) if no TOC exists

**Section Building** (`_build_sections`):
- Converts TOC entries into `Section` objects with page ranges
- Each section spans from its start page to the next section's start page (or document end)
- `_assign_section_paths` computes hierarchical path strings by tracking a stack of ancestors

**Table Processing**:
- `_extract_tables_pymupdf`: Uses PyMuPDF's built-in table finder (`page.find_tables()`)
- For each table, generates three text representations:
  - **Markdown** (`_table_to_markdown`): Pipe-delimited table with header separator
  - **Narration** (`_table_to_narration`): "In the row where Column1 is X, Column2 is Y" format
  - **CSV** (`_table_to_csv`): Comma-separated with semicolons escaped

**Main Extraction** (`extract_document`):
- Opens PDF with `pymupdf.open()`
- Extracts text per page, cleans it
- Builds section structure from TOC
- Assigns page text to sections by page range
- Attaches tables to their respective sections by page number
- Returns a fully populated `Document` object

**Batch Processing** (`extract_all_pdfs`):
- Discovers all `*.pdf` files in the input directory
- Calls `extract_document` for each
- Writes `{name}.txt` (full raw text) and `{name}_sections.json` (section metadata)
- Returns list of `Document` objects

---

### pipeline/chunk.py

Breaks extracted text into token-sized segments across multiple tiers.

**Data Class — `Chunk`**:
```python
@dataclass
class Chunk:
    id: str              # MD5 hash of "{pdf}:{tier}:{index}"
    text: str            # The chunk content
    token_count: int     # tiktoken count
    tier: str            # "micro", "standard", "section", "macro", "extra_overlap", "table"
    source_pdf: str      # Document name
    source_section: str  # Hierarchical path
    page_start: int
    page_end: int
```

**ID Generation** (`_make_id`):
```python
hashlib.md5(f"{source_pdf}:{tier}:{index}".encode()).hexdigest()[:12]
```
Deterministic — the same input always produces the same chunk ID. This is essential for the checkpoint/resume system.

**Core Chunking Algorithm** (`_chunk_text_by_tokens`):

```
Input: text string, target token size, overlap ratio

1. Split text into sentences (regex on . ! ? followed by space/newline)
2. Initialize current_chunk = [], current_tokens = 0
3. For each sentence:
   a. Count its tokens via tiktoken
   b. If sentence alone exceeds target:
      - Flush current chunk
      - Split sentence by words into target-sized pieces
   c. If adding sentence would exceed target:
      - Flush current chunk
      - Compute overlap: keep trailing sentences totaling overlap_ratio * target tokens
      - Start new chunk with those overlap sentences + current sentence
   d. Else: append sentence to current chunk
4. Flush final chunk
5. Filter: discard chunks < MIN_CHUNK_TOKENS or > MAX_CHUNK_TOKENS
```

The sentence-aware splitting prevents cutting in the middle of a thought. The overlap ensures context continuity between consecutive chunks.

**Table Chunking** (`_chunk_tables`):
- For each table in a section, creates up to 3 chunks (markdown, narration, CSV)
- Each format provides different training signal for the finetuned model
- Table chunks are tagged with `tier="table"`

**Main Function** (`chunk_document`):
1. For each tier in `CHUNK_SIZES`, runs `_chunk_text_by_tokens` on every section with that tier's target size and the standard overlap ratio
2. Runs an additional pass with `EXTRA_OVERLAP_RATIO` (0.5) at standard size -> `extra_overlap` tier
3. Runs `_chunk_tables` for table chunks
4. Returns a `dict[str, list[Chunk]]` mapping tier name to chunks (no file I/O here)

**Persistence** (`save_chunks`):
A separate function that takes the per-document chunk dict (`dict[doc_name, dict[tier, list[Chunk]]]`) and writes one JSONL file per (document, tier) pair: `{pdf_name}_{tier}.jsonl`. Returns the total chunk count for logging.

---

### pipeline/azure_client.py

Async Azure OpenAI client with production-grade rate limiting.

**Class — `AzureGenerationClient`**:

**Initialization**:
```python
self.client = AsyncAzureOpenAI(
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    api_version=os.getenv("AZURE_OPENAI_API_VERSION", "2025-01-01-preview"),
)
self.deployment = os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME", "synapt-dev-gpt-4.1")
self._semaphore = asyncio.Semaphore(MAX_CONCURRENT_REQUESTS)
self._rpm_tokens = REQUESTS_PER_MINUTE
self._rpm_lock = asyncio.Lock()        # serializes the sliding-window check
self._request_times: list[float] = []  # sliding window for RPM tracking
self.total_input_tokens = 0
self.total_output_tokens = 0
```

Both the deployment name and API version are env-driven with sensible defaults. The client and deployment are exposed as public attributes (`self.client`, `self.deployment`) — not underscored.

**Rate Limiting** (`_wait_for_rate_limit`):
```python
async def _wait_for_rate_limit(self):
    """Simple sliding-window RPM limiter."""
    async with self._rpm_lock:                  # serialize concurrent callers
        now = time.monotonic()
        # Purge timestamps older than 60 seconds
        self._request_times = [t for t in self._request_times if now - t < 60]
        if len(self._request_times) >= self._rpm_tokens:
            # Sleep until the oldest request falls out of the window
            wait_time = 60 - (now - self._request_times[0]) + 0.1
            if wait_time > 0:
                await asyncio.sleep(wait_time)
        self._request_times.append(time.monotonic())
```

This implements a sliding-window rate limiter. Rather than resetting a counter every 60 seconds (which causes burst/stall patterns), it tracks individual request timestamps and sleeps only when the window is full. The entire window check + append runs inside an `asyncio.Lock` (`self._rpm_lock`) so that multiple coroutines acquiring the semaphore concurrently can't all decide "the window has room" simultaneously and exceed the RPM cap.

**Retry Logic**:
Uses `tenacity` decorating the `generate` method directly:
- Retry on: `RateLimitError`, `APITimeoutError`, `APIConnectionError`
- Wait: `wait_exponential(multiplier=2, min=4, max=120)` — minimum 4 seconds on the first retry, then `2 * 2^(attempt-1)` seconds clamped to a 120-second ceiling. So effective waits are: 4s, 4s, 8s, 16s, 32s, 64s, 120s, 120s.
- Max attempts: 8
- A `before_sleep` hook logs each retry attempt with the exception that triggered it
- This handles transient Azure failures and 429 rate limit responses

**`generate(prompt, system, temperature, max_tokens) -> str`**:
1. Acquires semaphore (limits concurrent calls to `MAX_CONCURRENT_REQUESTS`)
2. Waits for rate limit clearance
3. Calls `_call_api` with retry
4. Tracks input/output token usage
5. Returns response text

**`generate_batch(items, system, temperature, max_tokens) -> list[str]`**:
- Wraps multiple prompts in `asyncio.gather()` calls to `generate()`
- The semaphore naturally limits concurrency within the batch
- Returns results in input order

---

### pipeline/prompts.py

Contains all LLM prompt templates and the strategy configuration registry. All prompts are domain-configurable via `config.DOMAIN`.

**System Prompt**: Built dynamically from `config.DOMAIN` and optional `config.DOMAIN_DESCRIPTION` via the `_build_system_prompt()` helper. The result is stored as the module-level constant `SYSTEM_PROMPT`, sent as the `system` message in every API call:
```python
def _build_system_prompt() -> str:
    if config.DOMAIN_DESCRIPTION:
        return (
            f"You are a {config.DOMAIN} domain expert and technical writer "
            f"{config.DOMAIN_DESCRIPTION}. "
            "You produce accurate, domain-rich content grounded in the source material provided."
        )
    return (
        f"You are a {config.DOMAIN} domain expert and technical writer. "
        "You produce accurate, domain-rich content grounded in the source material provided."
    )

SYSTEM_PROMPT = _build_system_prompt()
```

**Domain Parameterization**: Templates use the `__DOMAIN__` placeholder (chosen to avoid conflicts with Python's `{format_string}` syntax used for `{chunk_text}`, `{section_a}`, etc.). At module import time, all placeholders are resolved:
```python
_domain = config.DOMAIN
for _name in ["PARAPHRASE", "CONCEPT_EXPLANATION", "DEFINITIONS", ...]:
    globals()[_name] = globals()[_name].replace("__DOMAIN__", _domain)
```
This runs before `STRATEGY_CONFIG` is defined, so all strategy template references point to the already-resolved strings.

**Prompt Template Structure**:

Each template is a Python string with:
- **Context setting**: Tells the LLM what it's looking at (e.g., "the following __DOMAIN__ technical documentation")
- **Task description**: What transformation to perform
- **Output format**: Specifies `[MARKER]` headers to delimit each output section
- **Quality instructions**: "Preserve technical accuracy", "Use domain terminology", etc.
- **Placeholder**: `{chunk_text}`, `{section_a}`/`{section_b}`, or `{table_text}` for content injection

Example (simplified `paraphrase`, after domain resolution with `DOMAIN="telecommunications"`):
```
Rewrite the following telecommunications technical text in 10 different styles.
Output each as a separate section marked with the exact headers shown below.

[STYLE_1] Formal technical documentation
[STYLE_2] Conversational tutorial for a new engineer
...
[STYLE_10] Technical blog post style

Source text:
{chunk_text}

Output exactly 10 rewrites. Preserve all technical accuracy and domain-specific terminology.
```

**Strategy Configuration** (`STRATEGY_CONFIG`):

A dictionary mapping strategy names to their configuration:
```python
STRATEGY_CONFIG = {
    "paraphrase": {
        "template": PARAPHRASE_TEMPLATE,
        "markers": ["[STYLE_1]", "[STYLE_2]", ..., "[STYLE_10]"],
        "input_key": "chunk_text",
        "input_tier": "standard",
        "outputs_per_call": 10,
    },
    "cross_reference": {
        "template": CROSS_REFERENCE_TEMPLATE,
        "markers": ["[RELATIONSHIP]", "[DATA_FLOW]", "[OPERATIONAL]"],
        "input_key": None,  # uses section_a/section_b directly
        "input_tier": "macro",
        "outputs_per_call": 3,
    },
    # ... 12 more strategies
}
```

Key fields:
- `template`: The prompt string
- `markers`: The `[MARKER]` strings used to split the LLM response
- `input_key`: Which placeholder the chunk text goes into (or `None` for special handling)
- `input_tier`: Which chunk tier feeds this strategy
- `outputs_per_call`: Expected number of parsed segments per API response

---

### pipeline/generate.py

Orchestrates LLM calls across all strategies with batching and checkpointing.

**`run_all_strategies(chunks_by_tier, client, progress, output_dir) -> int`**:

The top-level generation coordinator:
1. Iterates through each strategy in `STRATEGY_CONFIG`
2. Selects chunks from the strategy's required tier
3. **Subsampling**: If a tier has >900 chunks, uses stride-based subsampling — `step = max(1, len(chunks) // 900)` and then `chunks[::step]`. This is deterministic (not random), preserving evenly-spaced coverage across the tier.
4. **Special cases**:
   - `cross_reference`: Pairs consecutive macro chunks as `(section_a, section_b)` by zipping `chunks[::2]` with `chunks[1::2]`
   - `table_narration`: Iterates all chunks in the `table` tier. Note: the current filter expression `[c for c in table_chunks if "table_md" in c.id or True]` short-circuits to always-true via `or True`, so all three table representations (markdown, narration, CSV) are sent. (Likely a bug — the apparent intent was to restrict to markdown-format table chunks only.)
5. Delegates to `run_strategy()`
6. Returns cumulative output count

**`run_strategy(strategy_name, chunks, client, progress, output_dir, chunks_b=None) -> int`**:

The per-strategy execution engine:

```
1. Load checkpoint: get list of already-completed chunk IDs
2. Filter pending: remove completed chunks from the work queue
3. If nothing pending: log skip, return 0

4. Open output file in append mode
5. Process in batches of MAX_CONCURRENT_REQUESTS (5):
   a. Build prompts:
      - _build_prompt(template, chunk, input_key)
      - Replaces {chunk_text} / {section_a} / {section_b} / {table_text}
   b. Call client.generate_batch(prompts) -> responses
   c. Parse each response:
      - _parse_response(response, markers)
      - Split on marker patterns using regex
      - Fallback: split on double newlines if markers not found
      - Filter segments: keep only those >= 20 characters
   d. Write parsed segments to JSONL with metadata
   e. Mark each chunk as done in progress tracker
   f. Checkpoint every CHECKPOINT_INTERVAL calls

6. Return total outputs generated
```

**Response Parsing** (`_parse_response`):

This is where LLM output becomes structured data:
```python
def _parse_response(response: str, markers: list[str]) -> list[str]:
    if not response or not response.strip():
        return []

    # Build regex: [STYLE_1]|[STYLE_2]|...|[STYLE_10]
    escaped = [re.escape(m) for m in markers]
    pattern = "|".join(escaped)

    # Split on markers — keep ALL parts (no preamble skip)
    parts = re.split(pattern, response)
    segments = []
    for part in parts:
        cleaned = part.strip()
        if cleaned and len(cleaned) > 20:  # strict greater-than
            segments.append(cleaned)

    # If any segments survived the threshold, return them
    if segments:
        return segments

    # Fallback: split on double newlines (only fires when no segments survived)
    paragraphs = [p.strip() for p in response.split("\n\n") if p.strip() and len(p.strip()) > 20]
    return paragraphs
```

The marker-based parsing is robust because:
- The LLM is explicitly instructed to use specific markers
- The regex handles variations in whitespace around markers
- The double-newline fallback catches cases where the LLM deviates from the format entirely (no usable segments produced)
- The 20-character strict-greater-than threshold filters out empty or trivial segments
- Note: the function does **not** discard the first split part — any preamble before the first marker is kept if it exceeds 20 characters

**Output JSONL Format**:
```json
{
  "text": "The parsed segment text...",
  "source_pdf": "Document_Name",
  "source_section": "Chapter > Section > Subsection",
  "strategy": "paraphrase",
  "parent_chunk_id": "93167c669094"
}
```

The `parent_chunk_id` field is `item["id"]` — the deterministic chunk ID for single-chunk strategies, or `"chunkA_chunkB"` for cross-reference pairs. This is the same value used for progress tracking via `mark_strategy_chunk_done()`, now also persisted in the output for downstream lineage grouping.

---

### pipeline/quality.py

Post-generation cleanup to ensure dataset quality.

**Domain Term Source**: `BASE_DOMAIN_TERMS` is set to `config.DOMAIN_SEED_TERMS` (empty by default). This replaces the previous hardcoded set of ~50 telecom-specific terms. When empty, the quality filter relies entirely on auto-extracted terms from the actual PDF content.

**Domain Term Extraction** (`extract_domain_terms`):
```python
def extract_domain_terms(raw_texts: list[str]) -> set[str]:
    # Frequency analysis using regex match for words/short phrases
    word_freq: dict[str, int] = {}
    for text in raw_texts:
        # Matches sequences of 3-31 alpha chars (with internal spaces), e.g.
        # "billing", "call detail record", "revenue management"
        words = re.findall(r'\b[A-Za-z][A-Za-z\s]{2,30}\b', text.lower())
        for w in words:
            w = w.strip()
            if len(w) > 2:
                word_freq[w] = word_freq.get(w, 0) + 1

    # Local set of common English stopwords to exclude
    common_words = {"the", "and", "for", ...}

    # Start with the config-driven seed set (empty by default)
    domain_terms = set(BASE_DOMAIN_TERMS)
    # Add terms appearing 3+ times that aren't common English and >3 chars
    for term, freq in word_freq.items():
        if freq >= 3 and term not in common_words and len(term) > 3:
            domain_terms.add(term)

    # Fallback: if too few terms extracted, lower threshold to prevent
    # the domain filter from becoming vacuous on small document sets
    if len(domain_terms) < 10:
        for term, freq in word_freq.items():
            if freq >= 2 and term not in common_words and len(term) > 3:
                domain_terms.add(term)

    return domain_terms
```

Key details:
- The regex pattern `\b[A-Za-z][A-Za-z\s]{2,30}\b` allows internal spaces, capturing multi-word phrases (e.g. "call detail record", "general ledger").
- The seed constant `BASE_DOMAIN_TERMS` comes from `config.DOMAIN_SEED_TERMS`. When empty (the default), the function relies entirely on frequency extraction from the actual source documents.
- The fallback threshold (frequency >= 2) activates when fewer than 10 terms are found, preventing cold-start failure on small document sets.

This builds a domain vocabulary dynamically from the source documents, adapting automatically to any domain without manual term lists.

**Exact Deduplication** (`deduplicate_exact`):
- Computes `md5(text.encode())` for each entry
- Keeps first occurrence of each hash
- O(n) time, O(n) space

**Fuzzy Deduplication** (`deduplicate_fuzzy`):
```python
def deduplicate_fuzzy(texts, threshold=0.85):
    if len(texts) < 2:
        return texts

    lsh = MinHashLSH(threshold=threshold, num_perm=MINHASH_NUM_PERM)
    minhashes = []

    # Pass 1: Build MinHash for every entry and insert into LSH
    for i, item in enumerate(texts):
        mh = _get_minhash(item["text"])  # 3-word shingles, num_perm=128
        minhashes.append(mh)
        try:
            lsh.insert(str(i), mh)
        except ValueError:
            # LSH refuses duplicate keys — ignore and continue
            pass

    # Pass 2: Query each entry; keep first occurrence, mark all later matches
    to_remove = set()
    for i, mh in enumerate(minhashes):
        if i in to_remove:
            continue
        result = lsh.query(mh)
        for r in result:
            r_idx = int(r)
            if r_idx != i and r_idx not in to_remove:
                to_remove.add(r_idx)

    return [item for i, item in enumerate(texts) if i not in to_remove]
```

MinHash LSH is the industry-standard approach for approximate nearest-neighbor deduplication at scale. The implementation uses a two-pass design:
1. **Pass 1** — convert each text into 3-word shingles, compute a 128-permutation MinHash signature, and insert all signatures into the LSH index (collecting them in `minhashes`).
2. **Pass 2** — for each entry, query the LSH for matches above the 0.85 threshold. The first occurrence is kept; later indexes that match it are added to `to_remove`. Already-removed indexes are skipped during querying.
3. Entries whose index ends up in `to_remove` are dropped.

This two-pass form (rather than insert-on-miss) ensures the LSH contains every candidate when each query runs — important because Jaccard similarity is symmetric and we want consistent membership decisions regardless of input order.

This is much faster than all-pairs comparison (O(n) vs O(n^2)) and catches paraphrases that differ in wording but are semantically near-identical.

**Quality Filtering** (`filter_low_quality`):

Applies five rejection criteria:
1. **Too short** (`< MIN_CHUNK_TOKENS`): Catches fragments and incomplete generations
2. **Too long** (`> MAX_CHUNK_TOKENS`): Catches runaway LLM outputs
3. **High special-char ratio** (`> MAX_SPECIAL_CHAR_RATIO`): Catches garbled text or code artifacts
4. **Low uniqueness** (`< 20% unique words`): Catches repetitive or degenerate outputs
5. **Off-domain** (no domain term present): Catches hallucinated content unrelated to the source material

Each rejection is counted by reason and logged for debugging.

**Pipeline** (`run_quality_pipeline`):
```python
def run_quality_pipeline(texts, domain_terms):
    after_exact = deduplicate_exact(texts)        # ~5-10% removed
    after_fuzzy = deduplicate_fuzzy(after_exact)   # ~10-15% removed
    after_filter = filter_low_quality(after_fuzzy, domain_terms)  # ~5% removed
    return after_filter
```

---

### pipeline/progress.py

Persistent state management for resumable execution.

**Class — `PipelineProgress`**:

**State Schema** (serialized to `state.json`):
```json
{
  "extraction_complete": false,
  "chunking_complete": false,
  "strategies": {
    "paraphrase": {
      "completed_ids": ["abc123", "def456", ...],
      "total_calls": 200,
      "total_outputs": 2000
    }
  },
  "quality_complete": false,
  "final_merge_complete": false
}
```

**Key Methods**:

- `__init__(progress_dir)`: Loads existing state from `state.json` or initializes empty state
- `save()`: Serializes state to JSON and writes to disk
- `extraction_complete` / `chunking_complete`: Properties with auto-save on set
- `mark_strategy_chunk_done(strategy, chunk_id, num_outputs)`: Adds chunk ID to completed set, increments counters
- `is_strategy_chunk_done(strategy, chunk_id) -> bool`: O(1) lookup via set membership
- `checkpoint()`: Explicit save, called every `CHECKPOINT_INTERVAL` API calls

The completed_ids list is the core of the resume mechanism. When a strategy resumes, it loads this list and skips any chunks already in it.

---

### pipeline/split.py

Lineage-aware dataset splitting. Groups rows by parent chunk lineage, assigns entire lineage families to train/validation/test splits with zero cross-split leakage.

**Class — `UnionFind`**:

A path-compressed disjoint set data structure for merging lineage groups. Used to handle the `cross_reference` strategy where two source chunks are paired — both must land in the same split.

```python
class UnionFind:
    def __init__(self):
        self.parent: dict[str, str] = {}

    def find(self, x: str) -> str:
        if x not in self.parent:
            self.parent[x] = x
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]  # path compression
            x = self.parent[x]
        return x

    def union(self, a: str, b: str):
        ra, rb = self.find(a), self.find(b)
        if ra != rb:
            self.parent[ra] = rb
```

**Lineage ID Computation** (`compute_lineage_id`):
- For rows with `parent_chunk_id`: returns it directly
- Fallback for legacy rows without `parent_chunk_id`: deterministic MD5 hash of `source_pdf:source_section:strategy`

**Lineage Grouping** (`_build_lineage_groups`):

Iterates all rows, computes lineage IDs, and builds connected components using UnionFind. For `cross_reference` rows (where `parent_chunk_id = "chunkA_chunkB"`), the pair ID and both constituent chunk IDs are unioned together. This ensures:
- All outputs from chunk A are in the same split
- All outputs from chunk B are in the same split
- All cross-reference outputs involving A or B are in the same split

The `_` separator in pair IDs is unambiguous because chunk IDs are 12-character hex strings (`[0-9a-f]` only).

**Split Assignment** (`assign_splits`):
1. Loads any previously frozen mapping (from `lineage_map.json`) and honors it
2. Sorts remaining group IDs lexicographically (for determinism regardless of dict iteration order)
3. Shuffles with `random.Random(seed)` for reproducibility
4. Assigns to splits by ratio using a greedy fill approach
5. Returns a `dict[str, str]` mapping group root ID to split name

**Output Row Enrichment** (`_build_output_rows`):

Transforms each quality-filtered row into the schema v2.0 format:
```json
{
  "schema_version": "2.0",
  "text": "...",
  "lineage_id": "93167c669094",
  "parent_chunk_id": "93167c669094",
  "source_id": "Document_Name",
  "source_type": "pdf",
  "source_path": "Chapter > Section",
  "generation_strategy": "paraphrase",
  "split": "train"
}
```

**Output Writers**:
- `_write_split_files(rows, output_dir)`: Writes `dataset_train.jsonl`, `dataset_validation.jsonl`, `dataset_test.jsonl`, and combined `dataset.jsonl`
- `_write_lineage_map(mapping, filepath)`: Persists the lineage group -> split mapping with metadata (seed, strategy, ratios) to `lineage_map.json`
- `_write_manifest(counts, groups, split_mapping, output_dir)`: Writes `dataset_manifest.json` with row counts, lineage counts, split config, and timestamp

**Top-Level Orchestrator** (`run_split_pipeline`):

Chains all steps: lineage grouping -> frozen mapping load -> split assignment -> row enrichment -> file writing -> lineage map persistence -> manifest generation. When `DATASET_SPLIT_ENABLED = False`, rows are still enriched with lineage metadata but all assigned to `"train"` and only a combined file is written.

The split engine does not participate in the checkpoint system because it is fast (in-memory, no API calls) and idempotent — it reruns from scratch each time to pick up any new data.

---

## Data Flow & Intermediate Formats

### Stage 1: Extracted Text

**File**: `output/extracted/{name}.txt`
```
Plain text, all pages concatenated, artifacts cleaned.
```

**File**: `output/extracted/{name}_sections.json`
```json
[
  {
    "title": "1 Introduction",
    "level": 1,
    "full_path": "1 Introduction",
    "page_start": 5,
    "page_end": 8,
    "text_length": 2847,
    "num_tables": 0
  }
]
```

### Stage 2: Chunks

**File**: `output/chunks/{name}_{tier}.jsonl`
```json
{"id": "93167c669094", "text": "...", "token_count": 244, "tier": "standard", "source_pdf": "Doc_Name", "source_section": "Chapter > Section", "page_start": 5, "page_end": 5}
```

### Stage 3: Generated Data

**File**: `output/generated/{strategy}.jsonl`
```json
{"text": "...", "source_pdf": "Doc_Name", "source_section": "Chapter > Section", "strategy": "paraphrase", "parent_chunk_id": "93167c669094"}
```

### Stage 4: Quality-Filtered (in-memory)

After merging raw + generated chunks and running dedup + quality filtering, rows are passed to the split engine as `list[dict]` with fields: `text`, `source_pdf`, `source_section`, `strategy`, `parent_chunk_id`.

### Stage 5: Final Dataset (schema v2.0)

**File**: `output/final/dataset_train.jsonl` / `dataset_validation.jsonl` / `dataset_test.jsonl` / `dataset.jsonl`
```json
{"schema_version": "2.0", "text": "...", "lineage_id": "93167c669094", "parent_chunk_id": "93167c669094", "source_id": "Doc_Name", "source_type": "pdf", "source_path": "Chapter > Section", "generation_strategy": "paraphrase", "split": "train"}
```

**File**: `output/final/dataset_manifest.json`
```json
{"dataset_name": "domain_dataset", "schema_version": "2.0", "split_strategy": "lineage", "seed": 42, "source_types": ["pdf"], "total_rows": 55000, "lineage_count": 4800, "train_rows": 49500, "validation_rows": 2750, "test_rows": 2750}
```

**File**: `output/final/lineage_map.json`
```json
{"schema_version": "2.0", "seed": 42, "strategy": "lineage", "ratios": {"train": 0.9, "validation": 0.05, "test": 0.05}, "mapping": {"93167c669094": "train", "44a8211b9a2c": "validation", ...}}
```

---

## Unsupervised Finetuning — Why This Approach

### What Is Unsupervised Finetuning?

Unsupervised finetuning (also called continued pretraining or domain-adaptive pretraining) involves training a pretrained language model on a domain-specific text corpus **without any labels or annotations**. The model learns to predict the next token in domain text, absorbing vocabulary, concepts, and reasoning patterns specific to the domain.

### Why Synthetic Data?

Raw source documents alone are insufficient for effective finetuning:

| Problem with Raw Docs | How Synthetic Generation Solves It |
|------------------------|-------------------------------------|
| Limited volume (hundreds of pages) | 14 strategies multiply each chunk into 2–10 outputs, producing 10–50x more text |
| Single writing style | Style variation strategies create diverse registers |
| Implicit knowledge | Analytical strategies (cross-reference, deep-dive) make implicit knowledge explicit |
| Structured data (tables) | Table narration converts structured data to natural language |
| No gradation of complexity | Concept explanation creates content at multiple expertise levels |
| Repetitive content (headers, boilerplate) | Chunking + quality filtering removes noise |

### Why Schema v2.0?

The final output uses a structured schema (v2.0) with 9 fields per row. While unsupervised finetuning typically only needs the `text` field, the additional metadata serves critical purposes:

- **`lineage_id` / `parent_chunk_id`**: Enable lineage-aware train/test splitting that prevents data leakage. Without these, a naive split could place semantically overlapping content in both training and evaluation sets.
- **`split`**: Each row is pre-assigned to `train`, `validation`, or `test`, so downstream consumers don't need to implement their own splitting logic.
- **`source_id` / `source_path`**: Provenance tracking — trace any output row back to its source document and section.
- **`generation_strategy`**: Enables analysis of which strategies contribute most to downstream performance.
- **`schema_version`**: Forward compatibility — consumers can detect and handle schema changes.

For finetuning frameworks that expect a bare `text` field, a simple `jq` extraction suffices: `jq '{text}' dataset_train.jsonl`.

---

## Rate Limiting & Concurrency Model

The pipeline manages three layers of concurrency control:

### Layer 1: Semaphore (Concurrency Cap)
```
asyncio.Semaphore(MAX_CONCURRENT_REQUESTS=5)
```
Ensures at most 5 API calls are in-flight simultaneously. This prevents overwhelming the Azure endpoint and keeps memory usage predictable.

### Layer 2: Sliding Window RPM
```
Track timestamps of last 60 seconds of requests.
If count >= REQUESTS_PER_MINUTE (30), sleep until oldest falls off.
```
This is smoother than a fixed-window counter. Instead of bursting 30 requests then waiting 60 seconds, it spreads requests evenly across time.

### Layer 3: Tenacity Retry
```
wait_exponential(multiplier=2, min=4, max=120)
  -> 4s, 4s, 8s, 16s, 32s, 64s, 120s, 120s
Max attempts: 8
Retries on: RateLimitError, APITimeoutError, APIConnectionError
```
Handles transient failures and Azure-side 429 responses that slip through the client-side rate limiter.

### How They Interact

```
generate_batch([prompt1, prompt2, prompt3, prompt4, prompt5])
  └── asyncio.gather(
        generate(prompt1),  # Each acquires semaphore (max 5 concurrent)
        generate(prompt2),  #   then checks RPM window
        generate(prompt3),  #     then calls API with retry
        generate(prompt4),
        generate(prompt5),
      )
```

The semaphore gates entry, the RPM tracker paces throughput, and tenacity handles failures. Together they produce smooth, reliable API utilization.

---

## Checkpoint/Resume System

The checkpoint system ensures no work is lost on interruption:

### What Gets Checkpointed
- Phase completion flags (extraction, chunking, quality, final_merge)
- Per-strategy: set of completed chunk IDs, total calls, total outputs

### When Checkpoints Happen
- After every `CHECKPOINT_INTERVAL` (100) API calls within a strategy
- When a phase completes
- On pipeline exit (implicit via progress property setters)

### How Resume Works

On restart:
1. `PipelineProgress.__init__` loads `state.json`
2. Phase 1/2: if `extraction_complete` / `chunking_complete` is true, skip entirely
3. Phase 3: for each strategy, `run_strategy` loads `completed_ids` and filters them out of the work queue
4. Phase 4: quality filtering reruns (fast, in-memory)
5. Phase 5: split engine reruns (fast, in-memory, idempotent) — if `DATASET_SPLIT_FREEZE_MAPPING` is true, previously assigned lineage groups retain their split assignments via `lineage_map.json`

This means a run interrupted after 150 of 500 paraphrase calls will resume at call 101 (the last checkpoint), re-doing at most 50 calls. The deterministic chunk IDs ensure the same chunks map to the same IDs across runs. The split engine does not need checkpointing because it processes all data in-memory from the quality-filtered output.

---

## Extending the Pipeline

### Adding a New Generation Strategy

1. **Define the template** in `pipeline/prompts.py` (use `__DOMAIN__` for domain-specific references):
   ```python
   COMPARISON_TEMPLATE = """Given this __DOMAIN__ documentation:
   ---
   {chunk_text}
   ---
   Compare the approach described here with industry standard practices.
   [INDUSTRY_STANDARD]
   [DIFFERENCES]
   [RECOMMENDATIONS]
   """
   ```
   Then add the template name to the domain resolution loop so `__DOMAIN__` is replaced at import time.

2. **Register in `STRATEGY_CONFIG`**:
   ```python
   "comparison": {
       "template": COMPARISON_TEMPLATE,
       "markers": ["[INDUSTRY_STANDARD]", "[DIFFERENCES]", "[RECOMMENDATIONS]"],
       "input_key": "chunk_text",
       "input_tier": "section",
       "outputs_per_call": 3,
   }
   ```

3. **Run the pipeline** — `run_all_strategies` automatically picks up the new strategy.

### Adding a New Chunk Tier

1. Add to `CHUNK_SIZES` in `config.py`:
   ```python
   CHUNK_SIZES = {"micro": 128, "standard": 256, "paragraph": 384, "section": 512, "macro": 1024}
   ```

2. Clear existing chunks and re-run — chunking regenerates all tiers.

### Custom Quality Rules

Add to `filter_low_quality` in `pipeline/quality.py`:
```python
# Example: reject chunks with too many acronyms
acronym_ratio = sum(1 for w in words if w.isupper() and len(w) > 1) / len(words)
if acronym_ratio > 0.3:
    rejection_counts["too_many_acronyms"] += 1
    continue
```

### Processing Different Domains

The entire pipeline is domain-agnostic. To switch domains, edit `config.py`:

```python
DOMAIN = "banking"                        # drives all prompts and system prompt
DOMAIN_DESCRIPTION = "specializing in retail banking and credit risk"  # optional
DOMAIN_SEED_TERMS = {"loan", "mortgage", "kyc", "aml"}                # optional
```

All LLM prompts, the system prompt, and quality filtering adapt automatically — no code changes required. If `DOMAIN_SEED_TERMS` is left empty, domain terms are auto-derived from the actual PDF content at runtime.
