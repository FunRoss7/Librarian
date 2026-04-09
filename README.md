# Librarian

A local, private reasoning tool for systems engineers. Not a search engine.

Librarian detects gaps, contradictions, and missing rationale across a corpus of engineering artifacts and customer discussions — and generates the next right question to ask. All inference runs on-device; no data leaves the machine.

The primary use case is **systems engineering work where the hard problem is not finding information but knowing what is missing**: requirements without documented rationale, design decisions without constraint coverage, questions that haven't been asked yet.

---

## The Problem With Standard RAG

Standard RAG retrieves documents similar to a query. That is sufficient for lookup questions but fails for systems engineering work, which requires:

- **Completeness checking** — "Do we have enough information to build a prototype?" requires detecting *absence*, which cosine similarity cannot do
- **Contradiction detection** — "Why wouldn't this simpler design work?" requires holding a proposed solution against documented constraints and finding conflicts
- **Rationale tracing** — "Did the customer ever explain why this is a requirement?" — requirements without rationale are brittle; the tool should flag them explicitly
- **Gap-driven question generation** — the most valuable output is often not an answer but the *next question*: one that targets a specific missing piece of information

---

## Core Insight

A hard query like "do we have enough to build a prototype?" can be decomposed into a set of smaller, answerable sub-queries. Each sub-query can be grounded against the corpus. The *compilation* of those results — including which sub-queries returned weak or no results — is where gap detection lives.

Weak retrieval on a sub-query is the gap signal. The system does not build a global knowledge graph upfront; it extracts structure at query time from already-retrieved content. Attempt the problem, go looking only when you hit a specific gap.

---

## North Star Behavior

An engineer asks:

> "Would a design that omits the secondary power bus work?"

The tool returns:

> "No. The customer specified in [artifact, date] that all subsystems must maintain operation during primary bus failure. No rationale was documented for this requirement. Suggested question for next customer meeting: 'Is the single-failure power isolation requirement driven by a specific mission scenario or a general reliability standard?'"

Grounded answer. Gap identified. Next question generated.

---

## Architecture

### Ingestion

```
source file
    │
    ▼
[file type?]
 text/PDF/DOCX ──► extract text ──────────────────┐
 image/sketch  ──► llava (Ollama) ──► prose desc  │
 audio/SRT     ──► Whisper / raw text ────────────┤
                                                  │
                                          high-fi text
                                                  │
                                                  ▼
                                              chunk text
                                                  │
                                                  ▼
                                        embedding model
                                           (Ollama)
                                                  │
                             ┌────────────────────┤
                             ▼                    ▼
                         SQLite               ChromaDB
                 (source metadata,    (chunk text + vectors,
                  high-fi text,        keyed to source_id)
                  project registry)
```

Four artifacts per source: original file, high-fidelity text, chunks, embeddings. The text translation is a first-class artifact, not a lossy preprocessing step.

---

### Query Pipeline

```
user query
    │
    ▼
┌─────────────────────────────────────┐
│        QUERY DECOMPOSITION          │
│                                     │
│  LLM decomposes into 3-5            │
│  atomic sub-queries                 │
│                                     │
│  consistency check:                 │
│  vector_sum(sub-queries) ≈ original │
│  (cosine similarity threshold)      │
│  → re-decompose if check fails      │
└────────────────┬────────────────────┘
                 │ sub-queries
                 ▼
┌─────────────────────────────────────┐
│        HyDE RETRIEVAL               │
│        (per sub-query)              │
│                                     │
│  LLM generates hypothetical         │
│  answer document for sub-query      │
│         │                           │
│         ▼                           │
│  embed hypothetical document        │
│  (document-shaped embeddings        │
│   navigate corpus better than       │
│   question-shaped embeddings)       │
│         │                           │
│         ▼                           │
│  cosine similarity → top-k chunks   │
│                                     │
│  ★ low confidence = gap signal      │
└────────────────┬────────────────────┘
                 │ retrieved chunks + gap flags
                 ▼
┌─────────────────────────────────────┐
│        SYNTHESIS                    │
│                                     │
│  compile all sub-query results      │
│  (including weak/empty results)     │
│                                     │
│  query-time structure extraction:   │
│  LLM identifies relationships,      │
│  contradictions, and absent         │
│  rationale from retrieved content   │
│                                     │
│  structured output:                 │
│  • Answer (grounded in evidence)    │
│  • Confidence                       │
│  • Gaps (weak sub-queries)          │
│  • Suggested Next Questions         │
└─────────────────────────────────────┘
```

**On gap signals:** A weak retrieval result has two possible causes — the information genuinely doesn't exist in the corpus, or retrieval failed despite it being present (vocabulary mismatch, unusual phrasing). HyDE reduces the second type of error but doesn't eliminate it. Human review is appropriate before acting on a gap.

---

## Build Phases

### Phase 1 — Grounded Retrieval (MVP)
**Goal:** Load artifacts, ask simple questions, get grounded answers with source citations.

- Set up Ollama with a base LLM
- Set up ChromaDB locally
- Build ingestion pipeline: PDF, DOCX, plain text → chunks → embeddings → vector store
- Build basic RAG query loop: embed query → retrieve top-k → LLM generates answer with citations
- Validate on real artifacts

**Deliverable:** Working local RAG over an engineering corpus.

---

### Phase 2 — Query Decomposition + Consistency Check
**Goal:** Break hard queries into sub-queries and validate the decomposition geometrically.

- Prompt the LLM to decompose a complex query into 3–5 atomic sub-queries
- Embed the original query and all sub-queries
- Compute vector sum of sub-query embeddings; check cosine similarity against original query embedding
- If similarity is below threshold, re-prompt for a different decomposition
- Run retrieval independently per sub-query; compile results grouped by sub-query

**Deliverable:** Decomposed query answering with per-sub-query evidence.

---

### Phase 3 — HyDE Integration
**Goal:** Improve retrieval quality by searching with hypothetical answer documents.

- For each sub-query, prompt LLM: "Write a short technical document that would answer this question"
- Embed the hypothetical document instead of the raw sub-query
- Use that embedding for corpus retrieval
- Compare retrieval quality against Phase 2 baseline on known test queries

**Deliverable:** Measurably improved retrieval on complex sub-queries.

---

### Phase 4 — Gap Detection and Query-Time Structure Extraction
**Goal:** Make absence visible; generate targeted follow-up questions.

- Flag sub-queries with weak or empty retrieval as explicit gaps
- At synthesis time, prompt LLM to extract typed relationships from retrieved chunks (supports, contradicts, requires, rationale-for) — structure extracted only from content already shown to be relevant
- Surface gaps explicitly in query responses: "This requirement has no documented rationale"
- Generate targeted follow-up questions for each gap

**Deliverable:** Gap report per query; suggested customer discussion agenda generated automatically.

---

### Phase 5 — Contradiction Detection
**Goal:** Answer "why wouldn't this simpler design work?" by checking a proposed design against corpus constraints.

- Accept a proposed design as input (text description or structured spec)
- Decompose and retrieve relevant constraints from the corpus
- Identify constraints that the proposed design violates or leaves unaddressed
- Return: specific constraints violated, source artifacts, rationale if present

**Deliverable:** Automated design review against corpus constraints.

---

## Open Decisions

### D1 — Framework: LlamaIndex vs. raw

**Question:** Build on LlamaIndex for orchestration primitives, or implement the pipeline directly against ChromaDB and Ollama?

**Stakes:** Highest-leverage decision. Determines how much boilerplate is needed to implement query decomposition, HyDE, and the synthesis loop cleanly.

**Options:**
- **LlamaIndex** — handles chunking, embedding, vector store, and Ollama integration. More control than PrivateGPT. Need to verify it doesn't fight the decomposition + HyDE pattern.
- **Raw** — direct calls to Ollama API + ChromaDB + SQLite. Maximum control, maximum build cost. Justified if LlamaIndex abstractions resist the custom query pipeline.

---

### D2 — Embedding model

**Question:** Which embedding model gives the best retrieval quality at acceptable CPU latency?

**Candidates:**
- `nomic-embed-text` via Ollama
- `all-MiniLM-L6-v2` via `sentence-transformers` — very fast on CPU
- `bge-small-en-v1.5` via `sentence-transformers` — consistently outperforms MiniLM on retrieval benchmarks

Note: the vector sum consistency check in Phase 2 may behave differently across embedding models — worth testing.

---

### D3 — LLM for CPU-only inference

**Question:** Which Ollama model gives the best quality/latency tradeoff with no GPU?

**Candidates:**
- `phi3:mini` (3.8B) — fast on CPU, strong reasoning for its size
- `llama3.2:3b` — good instruction following, smallest viable
- `mistral:7b-q4_K_M` — best quality, slowest on CPU
- `qwen2.5:3b` — competitive with llama3.2 at same size

The decomposition and HyDE steps require reliable instruction following; quality matters more here than in simple Q&A.

---

### D4 — Chunk size and overlap

**Question:** What chunk size and overlap produces the best retrieval on requirements-style documents?

**Candidates:** 256 / 512 / 1024 tokens × 10% / 25% overlap.

---

## Hardware Targets

| Config | LLM | Embedding | Notes |
|---|---|---|---|
| CPU only | `phi3:mini` or `llama3.2:3b` | `bge-small-en-v1.5` | Slow but functional; decomposition adds latency |
| 6GB GPU | `mistral:7b-q4_K_M` | `nomic-embed-text` | Comfortable fit; practical for daily use |

---

## Key Dependencies

- **Ollama** — local LLM and embedding model serving
- **ChromaDB** or **Qdrant** — local vector store
- **SQLite** — source metadata and high-fidelity text storage
- **LlamaIndex** — orchestration (pending D1)
- **FastAPI** — thin API layer for future UI integration
- **Python** — primary implementation language

**Stretch:**
- **Whisper** (`openai-whisper`) — audio transcription for meeting recordings
- **pyannote.audio** — speaker diarization
- **llava** via Ollama — whiteboard sketch and image ingestion
