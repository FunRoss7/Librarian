# Librarian

An offline-first RAG (Retrieval-Augmented Generation) system optimized for natural language queries over heterogeneous local sources. Designed for environments where cloud AI tools are not permitted — proprietary, regulated, or simply air-gapped.

The primary use case is **requirements extraction and validation from stakeholder discussions**: load meeting notes, transcripts, sketches, and documents into a named project, ask natural language questions, and get answers grounded in your actual source material with the originals surfaced for human verification.

---

## What Makes It Special

Most RAG tools are generic document Q&A. Librarian is designed around a specific, harder workflow:

**1. Analog-to-digital translation pipeline**
Non-text sources (images of whiteboard sketches, audio recordings, subtitle files) are translated to clean, high-fidelity prose before embedding. This translation is a first-class artifact stored alongside the original — not a lossy preprocessing step.

**2. Project scoping**
All sources and queries are isolated within a named project (e.g., `acme-requirements-q1-2026`). You are never querying across projects unless you explicitly ask to. This mirrors how knowledge work actually happens.

**3. The feasibility loop**
Beyond simple Q&A, Librarian supports a research query mode: given a question like *"why not use approach X?"*, the system retrieves candidate sources, filters them with the LLM for relevance, and presents the **original source excerpts** to the user — not a synthesized answer. The human makes the call; the LLM does the legwork.

**4. No cloud dependencies**
Every component — embedding, inference, storage — runs locally via Ollama. No data leaves the machine. Suitable for proprietary and regulated environments.

---

## Notional Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        INGESTION                        │
│                                                         │
│  source file                                            │
│      │                                                  │
│      ▼                                                  │
│  [file type?]                                           │
│   text/PDF/DOCX ──► extract text ──────────────────┐   │
│   image/sketch  ──► llava (Ollama) ──► prose desc  │   │
│   audio/SRT     ──► Whisper / raw text ────────────┤   │
│                                                    │   │
│                                            high-fi text │
│                                                    │   │
│                                            ▼           │
│                                        chunk text       │
│                                      (400 tok / 50 overlap)
│                                            │           │
│                                            ▼           │
│                                  nomic-embed-text       │
│                                     (Ollama)            │
│                                            │           │
│                                       embeddings        │
│                                            │           │
│              ┌─────────────────────────────┤           │
│              ▼                             ▼           │
│          SQLite                        ChromaDB         │
│  (source metadata,              (chunk text + vectors,  │
│   high-fi text,                  keyed to source_id)    │
│   project registry)                                     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                      QUERY (RAG)                        │
│                                                         │
│  user query                                             │
│      │                                                  │
│      ▼                                                  │
│  nomic-embed-text ──► query vector                      │
│      │                                                  │
│      ▼                                                  │
│  ChromaDB cosine similarity ──► top-k chunks            │
│      │                                                  │
│      ▼                                                  │
│  fetch high-fi text for matched sources (SQLite)        │
│      │                                                  │
│      ▼                                                  │
│  LLM prompt: [context excerpts] + [query] ──► answer    │
│      │                                                  │
│      ▼                                                  │
│  present original source files to user                  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   FEASIBILITY LOOP                      │
│                                                         │
│  "why not X?" query                                     │
│      │                                                  │
│      ▼                                                  │
│  similarity search ──► top-20 candidate chunks          │
│      │                                                  │
│      ▼                                                  │
│  single LLM call: rank/filter candidates by relevance   │
│      │                                                  │
│      ▼                                                  │
│  top-5 results with LLM relevance summary               │
│      │                                                  │
│      ▼                                                  │
│  present original source excerpts to user               │
│  (user decides — LLM does not conclude)                 │
└─────────────────────────────────────────────────────────┘
```

**Database schema (SQLite)**

```
projects    id, name, description, created_at
sources     id, project_id, original_path, file_type, ingested_at
documents   id, source_id, high_fi_text
```

**ChromaDB**: one collection per project, each record holds chunk text + embedding vector + source_id foreign key.

---

## Open Decisions

### D1 — Framework: LlamaIndex vs. PrivateGPT vs. raw

**Question:** Should we build on LlamaIndex (framework), adopt PrivateGPT (full application), or implement the RAG pipeline directly against ChromaDB and Ollama?

**Stakes:** This is the highest-leverage decision. It determines development velocity, how much control we have over the feasibility loop, and long-term maintainability.

**Options:**
- **PrivateGPT** — ships with ingestion, embedding, Ollama integration, project workspaces, and a REST API. Low build cost, but the feasibility loop would be bolted on via its API and may hit abstraction limits.
- **LlamaIndex** — handles chunking, embedding, vector store, and Ollama integration as a framework. More boilerplate than PrivateGPT, but full control over every stage including the feasibility loop.
- **Raw** — direct calls to Ollama API + ChromaDB + SQLite. Maximum control, maximum build cost. Justified only if frameworks prove too constraining.

→ *See Experiment 1 and Experiment 2*

---

### D2 — Embedding model

**Question:** Which embedding model gives the best retrieval quality at acceptable CPU latency?

**Stakes:** Embedding quality directly determines retrieval quality. A poor embedding model means relevant sources get missed regardless of LLM quality.

**Candidates:**
- `nomic-embed-text` via Ollama — consistent with keeping everything in the Ollama stack
- `all-MiniLM-L6-v2` via `sentence-transformers` — very fast on CPU, well-benchmarked
- `bge-small-en-v1.5` via `sentence-transformers` — consistently outperforms MiniLM on retrieval tasks

→ *See Experiment 3*

---

### D3 — LLM selection for CPU-only inference

**Question:** Which Ollama model gives the best quality/latency tradeoff with no GPU?

**Stakes:** On CPU, a 7B model may take 30–90 seconds per response. Smaller models respond faster but may produce lower quality translations and summaries.

**Candidates:**
- `phi3:mini` (3.8B) — Microsoft, strong reasoning for its size, fast on CPU
- `llama3.2:3b` — Meta, good instruction following, smallest viable
- `mistral:7b-q4_K_M` — best quality of the group, slowest on CPU
- `qwen2.5:3b` — strong multilingual, competitive with llama3.2 at same size

→ *See Experiment 3*

---

### D4 — Chunk size and overlap

**Question:** What chunk size and overlap produces the best retrieval on requirements-style documents (meeting notes, transcripts, spec documents)?

**Stakes:** Too small and chunks lose context; too large and dissimilar content gets bundled into one embedding, reducing precision.

**Candidates to test:** 256 / 512 / 1024 tokens, with 10% and 25% overlap.

→ *See Experiment 4*

---

### D5 — Feasibility loop: single-pass vs. two-stage filtering

**Question:** Should the feasibility loop filter results with one LLM call over all candidates, or run two stages (embed filter → LLM rerank)?

**Stakes:** On CPU, multiple LLM calls compound. A single batch LLM call over top-20 chunks is likely faster and good enough, but two-stage may give higher precision.

→ *See Experiment 5*

---

## Research Regimen

The goal of these experiments is to make the open decisions above with evidence, not intuition. All experiments run CPU-only on a standard laptop. Each experiment produces a written finding that closes one decision.

---

### Experiment 1 — PrivateGPT evaluation

**Closes:** D1 (partial)

**Setup:**
1. Install PrivateGPT with Ollama backend (`pip install private-gpt`)
2. Configure to use `nomic-embed-text` for embeddings and `mistral:7b-q4` for inference
3. Prepare a sample dataset: 3–5 documents representing a realistic stakeholder meeting — e.g., a meeting transcript (plain text), a requirements doc (PDF), and a sketch description (manually written prose standing in for llava output)

**Tasks to evaluate:**
- Ingest all documents into a named workspace
- Query: *"What performance requirements were mentioned?"*
- Query: *"Who expressed concern about the timeline?"*
- Query: *"Are there any conflicting requirements between the two documents?"*
- Attempt to implement the feasibility loop via the PrivateGPT REST API

**Record:**
- Response quality (0–5 subjective rating per query)
- Ingestion time
- Query latency
- How much custom code was needed for the feasibility loop
- Blockers or abstraction limits hit

---

### Experiment 2 — LlamaIndex evaluation

**Closes:** D1 (partial), compare with Experiment 1

**Setup:**
1. `pip install llama-index llama-index-llms-ollama llama-index-embeddings-ollama chromadb`
2. Build a minimal RAG pipeline: ingest same sample dataset, project-scoped ChromaDB collection, query interface
3. Implement the feasibility loop: retrieve top-20, single LLM filter call, return top-5 with originals

**Tasks to evaluate:** Same queries as Experiment 1 for direct comparison.

**Record:** Same metrics. Additional: lines of code to implement feasibility loop. Points where LlamaIndex abstractions helped vs. fought the implementation.

**Decision rule:** If LlamaIndex feasibility loop implementation is clean (< ~150 lines, no major workarounds) and query quality is comparable to PrivateGPT, prefer LlamaIndex for control. If PrivateGPT REST API supports the feasibility loop cleanly, prefer it for lower maintenance burden.

---

### Experiment 3 — Embedding model and LLM benchmarking

**Closes:** D2, D3

**Setup:**
1. Fix the framework (use result of Experiments 1–2)
2. Prepare a retrieval test set: 10 queries with known ground-truth relevant documents from the sample dataset
3. Test each embedding model: `nomic-embed-text`, `all-MiniLM-L6-v2`, `bge-small-en-v1.5`
4. For each embedding model, record: recall@5 (how many of the 5 retrieved chunks contain the ground-truth answer), indexing time, query embedding latency

**For LLM benchmarking:**
1. Fix the best embedding model from above
2. Test `phi3:mini`, `llama3.2:3b`, `mistral:7b-q4_K_M`, `qwen2.5:3b`
3. For each: run the 10 queries, rate answer quality (0–5), record time-to-first-token and total response time on CPU

**Decision rule:** Pick the embedding model with highest recall@5. For LLM, if `phi3:mini` or `llama3.2:3b` scores within 1 point of `mistral:7b-q4` on average quality, prefer the smaller model for latency. If `mistral:7b` quality is clearly better, use it and accept the latency.

---

### Experiment 4 — Chunk size optimization

**Closes:** D4

**Setup:**
1. Re-ingest the sample dataset 6 times: chunks of 256/512/1024 tokens × 10%/25% overlap
2. Run the same 10 ground-truth queries against each index
3. Record recall@5 for each configuration

**Decision rule:** Pick the configuration with highest recall@5. In case of tie, prefer smaller chunks (faster embedding, lower context window usage at query time).

---

### Experiment 5 — Feasibility loop design

**Closes:** D5

**Setup:**
1. Use the best configuration from Experiments 1–4
2. Prepare 5 feasibility queries (e.g., *"Why not build this as a microservice?"*, *"Why not use a relational database for storage?"*)
3. Implement and compare:
   - **Single-pass:** retrieve top-20 chunks, one LLM call with all 20, return top-5
   - **Two-stage:** retrieve top-20 chunks, LLM call to score each independently, return top-5

**Record:** Total latency end-to-end, quality of filtered results (subjective 0–5), number of LLM calls.

**Decision rule:** If single-pass latency is less than 2× two-stage and quality is within 1 point, use single-pass. Fewer LLM calls is strongly preferred on CPU.

---

## Stretch Goals

- **Audio transcription** — local Whisper (`openai-whisper`, `base` model on CPU) for meeting recordings
- **Speaker diarization** — `pyannote.audio` to attribute transcript lines to speakers; requires HuggingFace model download, runs locally
- **Vision ingestion** — `llava` via Ollama to describe whiteboard sketches and handwritten notes; requires model swap on 6GB GPU (not relevant for CPU-only setup)
- **Cross-project search** — explicit mode to query across all projects simultaneously

---

## Hardware Targets

| Config | LLM | Embedding | Notes |
|---|---|---|---|
| CPU only | `phi3:mini` or `llama3.2:3b` | `bge-small-en-v1.5` | Slow but functional; ~30–60s per query |
| 6GB GPU | `mistral:7b-q4_K_M` | `nomic-embed-text` | Comfortable fit; ~3–5s per query |
| 6GB GPU + vision | `llava:7b-q4` for ingestion, `mistral:7b-q4` for query | `nomic-embed-text` | Ollama swaps models; image ingestion slow |
