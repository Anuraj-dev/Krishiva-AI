# Krishiva AI

> A farmer's conversational crop-advisory assistant. Ask about a crop problem in
> plain language — English, Hindi, Bengali, or Assamese — and get a grounded,
> cited answer with a confidence-scored diagnosis, drawn from a curated
> agricultural knowledge base rather than an open-ended model guess.

**Status: MVP v1.** The knowledge base currently covers **six rice diseases**
with a regional focus on **Assam / Northeast India**. Everything runs locally —
retrieval, reranking, and generation can all happen on your machine with no
external API required.

---

## What it does

A farmer describes what they see — *"yellow diamond-shaped spots on my paddy
leaf after the foggy nights"* — and Krishiva:

1. **Detects the language** of the query (en / hi / bn / as).
2. **Retrieves** the most relevant knowledge chunks using hybrid search (dense
   vectors + BM25) with cross-encoder reranking.
3. **Derives a diagnosis** deterministically from the retrieval evidence —
   *problem + a 0–1 confidence score* — so the confidence shown is grounded in
   the retrieved sources, not the model's self-assessment.
4. **Generates a grounded answer** with an LLM, constrained to the retrieved
   knowledge, in the farmer's language.
5. **Returns citations** to the exact knowledge entries used.

The knowledge is authored for how farmers actually speak: each disease entry
carries farmer phrases and analogies in four languages, disambiguation trees
("is it Blast or Brown Spot?"), scientific symptoms, region-specific seasonal
risk calendars, and concrete management steps (cultural, biological, chemical
with doses).

---

## Architecture

Krishiva is a monorepo with three cleanly separated layers:

```
┌──────────────┐     POST /api/v1/query      ┌──────────────┐        ┌────────────────┐
│   frontend   │ ──────────────────────────▶ │   backend    │ ─────▶ │     engine     │
│  Next.js 16  │ ◀────────────────────────── │   FastAPI    │ ◀───── │  RAG pipeline  │
│  React 19    │   { answer, diagnosis,      │              │        │                │
└──────────────┘     sources, language }     └──────────────┘        └───────┬────────┘
                                                                             │
                                              ┌──────────────────────────────┼──────────────┐
                                              │                              │              │
                                       ┌──────▼──────┐              ┌────────▼───────┐  ┌───▼────┐
                                       │  Retrieval  │              │   Generation   │  │   KB   │
                                       │ dense+BM25  │              │  LLM / extract │  │  JSON  │
                                       │  + reranker │              │  + diagnosis   │  │        │
                                       └──────┬──────┘              └────────────────┘  └────────┘
                                       ┌──────▼───────┐
                                       │ Qdrant (emb) │
                                       │ vector store │
                                       └──────────────┘
```

- **`engine/`** — the knowledge engine. A self-contained RAG pipeline with no
  backend dependency: knowledge-base loading/validation, indexing, hybrid
  retrieval + reranking, deterministic diagnosis, and LLM generation.
- **`backend/`** — a thin FastAPI service. Exposes `POST /api/v1/query`, adapts
  the engine's DTOs to a stable HTTP wire contract, and degrades gracefully to a
  deterministic stub engine if the real one can't load.
- **`frontend/`** — a Next.js web client (chat UI, diagnosis cards, source
  citations, language picker, farm profile). See
  [`frontend/README.md`](frontend/README.md) for details.
- **`data/`** — the knowledge bases (`data/knowledge/`) and the built vector
  index (`data/vector_store/`).

### The engine pipeline

```
query
  → detect language
  → hybrid retrieve (dense BGE embeddings + BM25 sparse, RRF fusion)
  → cross-encoder rerank (bge-reranker-base)
  → problem-diversity cap (so competing diagnoses surface)
  → derive diagnosis (deterministic heuristic over retrieval scores)
  → assemble prompt from retrieved chunks
  → LLM synthesises a grounded answer
  → EngineAnswer (answer + language + diagnosis + citations)
```

### LLM providers

The generator is swappable via `KRISHIVA_LLM_PROVIDER`:

| Provider        | What it uses                                              |
|-----------------|----------------------------------------------------------|
| `openai_compat` | A local OpenAI-compatible server — **LM Studio** (default), Ollama, or vLLM. This is the default for local dev (Qwen3 4B). |
| `anthropic`     | The Claude API (needs `ANTHROPIC_API_KEY`).              |
| `extractive`    | No LLM. Stitches an answer directly from the retrieved knowledge — keeps the app functional offline / with no key. |
| `auto`          | Anthropic if a key is set, otherwise `extractive`.       |

If a configured provider fails to initialise, the engine logs a warning and
falls back to the extractive generator rather than crashing.

---

## Tech stack

**Backend / engine (Python 3.14, managed with [uv](https://docs.astral.sh/uv/))**
- FastAPI + Uvicorn
- Pydantic / pydantic-settings
- sentence-transformers (`BAAI/bge-small-en-v1.5` embeddings, `BAAI/bge-reranker-base` reranker)
- Qdrant (embedded, on-disk) vector store
- `rank-bm25` for sparse retrieval
- Anthropic SDK / httpx (for local OpenAI-compatible servers)
- loguru, orjson

**Frontend (Node)**
- Next.js 16 (App Router) + React 19 + TypeScript
- Tailwind CSS v4, Zustand, TanStack Query
- react-markdown, lucide-react

---

## Getting started

### Prerequisites

- **Python 3.14** and **[uv](https://docs.astral.sh/uv/)**
- **Node.js** (for the frontend)
- *(Optional)* **[LM Studio](https://lmstudio.ai/)** running a local model
  (e.g. Qwen3 4B) for full conversational answers. Without it, the engine uses
  the extractive fallback.

### 1. Configure

```bash
cp .env.example .env
```

The defaults target a local LM Studio server at `http://localhost:1234/v1`. To
run with no LLM at all, set `KRISHIVA_LLM_PROVIDER=extractive`.

### 2. Install dependencies & build the vector index

```bash
uv sync
uv run python scripts/build_index.py
```

The first run downloads the embedding and reranker models and builds the Qdrant
index from `data/knowledge/`. This is an offline step — don't run it while the
backend is holding the same store open (the embedded store is single-writer).

### 3. Run the backend

```bash
uv run python main.py          # → http://127.0.0.1:8000
```

Interactive API docs: <http://127.0.0.1:8000/docs>.

### 4. Run the frontend

```bash
cd frontend
npm install
cp .env.example .env.local     # points NEXT_PUBLIC_API_BASE_URL at the backend
npm run dev                    # → http://localhost:3000
```

---

## API

### `GET /health`

Liveness probe. Returns `{ "status": "ok", "version": "..." }`.

### `POST /api/v1/query`

Answer a single farmer query.

**Request**

```json
{
  "query": "yellow diamond spots on my paddy leaf after foggy nights",
  "language": "en",
  "region": "Assam",
  "crop": "rice"
}
```

`language`, `region`, and `crop` are optional. When `language` is omitted the
engine detects it; a non-matching `crop` hint is ignored rather than returning
nothing.

**Response**

```json
{
  "query": "yellow diamond spots on my paddy leaf after foggy nights",
  "language": "en",
  "answer": "This looks like Rice Blast…",
  "diagnosis": { "problem": "Rice Blast", "confidence": 0.87 },
  "sources": [
    { "id": "blast_farmer_entry_leaf_001", "title": "…", "score": 0.94 }
  ],
  "request_id": "…"
}
```

Errors are returned as a structured envelope:
`{ "error": { "code", "message", "request_id" } }`.

---

## Knowledge coverage (v1)

Six rice diseases, each a multi-chunk JSON knowledge base with multilingual
farmer phrases, disambiguation logic, scientific symptoms, Assam-specific
seasonal context, and management protocols:

| Disease        | Pathogen / type            |
|----------------|----------------------------|
| Rice Blast     | *Magnaporthe oryzae* (fungal) |
| Brown Spot     | Fungal leaf spot           |
| False Smut     | Fungal grain disease       |
| Sheath Blight  | Fungal                     |
| Sheath Rot     | Fungal                     |
| Stem Rot       | Fungal                     |

**Languages:** English (`en`), Hindi (`hi`), Bengali (`bn`), Assamese (`as`).

Sources cited in the knowledge base include ICAR-AAU Jorhat, IRRI, CRRI Cuttack,
and SAU Assam.

---

## Configuration reference

All settings are environment variables (see [`.env.example`](.env.example)):

| Variable | Default | Purpose |
|----------|---------|---------|
| `KRISHIVA_LLM_PROVIDER` | `openai_compat` | `auto` / `anthropic` / `openai_compat` / `extractive` |
| `KRISHIVA_LLM_BASE_URL` | `http://localhost:1234/v1` | Local model server (LM Studio / Ollama / vLLM) |
| `KRISHIVA_LLM_MODEL` | `qwen/qwen3-4b` | Model id as reported by your server |
| `KRISHIVA_LLM_MAX_TOKENS` | `1500` | Generation cap |
| `KRISHIVA_LLM_DISABLE_THINKING` | `1` | Inject `/no_think` for reasoning models (Qwen3) |
| `ANTHROPIC_API_KEY` | — | Required only for `provider=anthropic` |
| `KRISHIVA_EMBEDDING_MODEL` | `BAAI/bge-small-en-v1.5` | Dense embedding model |
| `KRISHIVA_RERANKER_MODEL` | `BAAI/bge-reranker-base` | Cross-encoder reranker (empty to disable) |
| `KRISHIVA_TOP_K` | `6` | Chunks returned per query |
| `KRISHIVA_USE_REAL_ENGINE` | `1` | Set `0` to force the deterministic stub engine |
| `KRISHIVA_LOG_LEVEL` | `INFO` | Log verbosity |
| `KRISHIVA_CORS_ORIGINS` | `["*"]` | Allowed CORS origins |

---

## Development

```bash
# Tests
uv run pytest

# Lint / format / type-check
uv run ruff check .
uv run ruff format .
uv run mypy .
```

The `scripts/` directory holds standalone verification scripts for each engine
layer (loader, parser, validator, embedder, retriever, formatter, backend,
end-to-end), useful for debugging a single stage in isolation — e.g.
`uv run python scripts/verify_engine.py`.

---

## Project layout

```
krishiva-ai/
├── engine/               # RAG pipeline (retrieval, generation, knowledge base)
│   ├── knowledge_base/   #   loading, parsing, validation of KB JSON
│   ├── indexing/         #   builds the vector index
│   ├── retrieval/        #   embedder, vector store, BM25, reranker, formatting
│   ├── generation/       #   language detection, diagnosis, prompt, LLM clients
│   └── pipeline.py       #   ties it all together (KrishivaEngine)
├── backend/              # FastAPI service
│   ├── api/              #   routes + dependencies
│   ├── core/             #   config, logging, middleware, error handling
│   ├── engine_gateway/   #   adapter to the engine + deterministic stub
│   ├── schemas/          #   HTTP request/response models
│   └── services/         #   query service
├── frontend/             # Next.js web client
├── data/
│   ├── knowledge/rice/   # the six rice-disease knowledge bases
│   └── vector_store/     # built Qdrant index
├── scripts/              # build_index + per-layer verification scripts
├── tests/                # pytest suite
└── main.py               # dev entrypoint (uvicorn)
```

---

## Roadmap (beyond v1)

- More crops and problem categories (pests, nutrient deficiencies, weather)
- Token streaming for the chat UI
- Image-based diagnosis (photo upload)
- Server-side conversation history and farm profiles (currently client-side)
- Broader language coverage in the UI catalog
