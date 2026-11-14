<p align="center">
  <img src="frontend/src/app/icon.svg" alt="Krishiva AI logo" width="88" height="88" />
</p>

<h1 align="center">Krishiva AI</h1>

<p align="center">
  A crop advisory chat for farmers in English, Hindi, Bengali and Assamese.<br />
  Each answer cites the knowledge it used and comes with a confidence-scored diagnosis.
</p>

<p align="center">
  <a href="#running-it">Run it locally</a>
  ·
  <a href="#api">API</a>
  ·
  <a href="#configuration">Configuration</a>
</p>

Krishiva answers a farmer's question about a crop problem. You describe what you see on the plant in plain words. Krishiva searches a knowledge base for matching chunks, names the most likely problem with a 0 to 1 confidence score, and writes the answer from those chunks. The confidence score comes from the retrieval scores, not from the language model.

It has three parts: a Python knowledge engine, a FastAPI backend that serves `POST /api/v1/query`, and a Next.js web client. Retrieval and reranking run on your machine. The answer step uses a local OpenAI-compatible server, the Anthropic API, or no model at all. You need Python 3.14 with uv to run the backend and Node.js to run the web client.

## What you get

- **Hybrid search with reranking.** Dense BGE embeddings and BM25 run in one Qdrant query, fused with reciprocal rank fusion. A cross-encoder rescores the top 20 candidates. Each problem contributes up to three chunks before the rest fill in, so competing diagnoses stay visible.
- **A confidence score from the evidence.** The score blends three signals: the top match's reranker probability, its margin over the best competing problem, and the share of retrieved chunks that agree with it. The model does not set the score. If the top match scores below 0.15, the response has no diagnosis.
- **Language from the query.** Devanagari text is answered in Hindi. Bengali script is Bengali, unless the text contains ৰ or ৱ, which only Assamese uses. Everything else is English. A `language` field in the request overrides detection.
- **Crop hints that never block a search.** A `crop` value filters the index to that crop. If nothing matches, the search runs again without the filter.
- **Answers without a model.** The `extractive` provider builds the answer from the retrieved text, up to 900 characters. It also takes over when the configured provider fails to start.
- **Sources and request IDs.** Each response lists up to five sources with an id, title and score. Every request gets an ID. It comes back in the response body and in the `X-Request-ID` header, and the server log lines for that request carry it too.
- **A chat client that keeps data in the browser.** Conversations, the farm profile and UI settings are saved in localStorage under `krishiva.conversations`, `krishiva.farm` and `krishiva.ui`. The backend keeps no conversation history.

## Running it

Krishiva is not packaged. You run the backend and the web client from a clone of this repo.

### Prerequisites

- Python 3.14 and [uv](https://docs.astral.sh/uv/)
- Node.js with npm
- Optionally, [LM Studio](https://lmstudio.ai/) with a model loaded and its local server running

### 1. Configure

```sh
cp .env.example .env
```

`.env.example` selects the `openai_compat` provider, so the backend expects a model server at `http://localhost:1234/v1`, which is LM Studio's default. Load `qwen/qwen3-4b` in LM Studio and start its server, or set `KRISHIVA_LLM_MODEL` to the id your server reports. To run without any model server, set `KRISHIVA_LLM_PROVIDER=extractive`. If the model server is down when a query arrives, the API returns `502` with the code `engine_error`.

### 2. Build the knowledge index

```sh
uv sync
uv run python scripts/build_index.py
```

The repository does not include knowledge base files. Put your JSON knowledge bases under `data/knowledge/`, at any depth, or point `KRISHIVA_KB_DIR` at them. The build reads every `*.json` file there, embeds the chunks and writes the Qdrant store to `data/vector_store/`. Without any JSON files, the script stops with `No knowledge JSON files found`. The build downloads the embedding model on first run. The reranker downloads on the first query.

Stop the backend before you rebuild. The embedded Qdrant store allows one writer.

### 3. Start the backend

```sh
uv run python main.py
```

The backend listens on `http://127.0.0.1:8000` and reloads on code changes. The interactive API docs are at `http://127.0.0.1:8000/docs`.

### 4. Start the web client

```sh
cd frontend
npm install
npm run dev
```

The client serves `http://localhost:3000` and calls the backend at `http://127.0.0.1:8000`. To point it at another backend, set `NEXT_PUBLIC_API_BASE_URL` in `frontend/.env.local`.

## How it works

`engine/` is the knowledge engine. It has no dependency on the backend. `backend/` adapts the engine to the HTTP contract. `frontend/` is the web client. A knowledge base is a JSON file of chunks. Each chunk has one type: farmer entry, scientific symptom, management, disambiguation, regional context or synonym expansion.

A query goes through these steps:

1. Detect the language from the script, or use the request's hint.
2. Embed the query with `BAAI/bge-small-en-v1.5` and encode it as BM25 terms.
3. Run one Qdrant query over both vectors and fuse the results with reciprocal rank fusion. Keep the top 20.
4. Rerank with `BAAI/bge-reranker-base`, then cap each problem at three chunks.
5. Derive the diagnosis from the chunk scores. No model call is involved.
6. Build the prompt from the chunks, the diagnosis and the question, then send it to the answer provider.
7. Return the answer, the language, the diagnosis and up to five sources.

The system prompt limits the assistant to smallholder farmers in Assam and Northeast India. It tells the model to answer only from the retrieved chunks and to say so when they do not cover the question.

## Configuration

Every setting is an environment variable. Keep `.env` in the directory you run the commands from, which is the repo root. The defaults below come from the code. `.env.example` overrides some of them for LM Studio.

| Variable | Default | Purpose |
| --- | --- | --- |
| `KRISHIVA_LLM_PROVIDER` | `auto` | `auto`, `anthropic`, `openai_compat` or `extractive`. `auto` uses `anthropic` when `ANTHROPIC_API_KEY` is set and `extractive` otherwise. |
| `KRISHIVA_LLM_BASE_URL` | `http://localhost:1234/v1` | Endpoint for `openai_compat`. Works with LM Studio, Ollama and vLLM. |
| `KRISHIVA_LLM_MODEL` | `claude-opus-4-8` | Model id. Set it to the id your local server reports. `.env.example` sets `qwen/qwen3-4b`. |
| `KRISHIVA_LLM_API_KEY` | `lm-studio` | Bearer token sent to the local server. |
| `KRISHIVA_LLM_MAX_TOKENS` | `1200` | Output token cap for both providers. |
| `KRISHIVA_LLM_DISABLE_THINKING` | `true` | Adds `/no_think` to the system prompt for reasoning models such as Qwen3. |
| `KRISHIVA_USE_LLM` | `true` | Set to `0` to skip the model and use `extractive`. |
| `ANTHROPIC_API_KEY` | none | Required for the `anthropic` provider. |
| `KRISHIVA_KB_DIR` | `data/knowledge` | Where `build_index.py` looks for knowledge JSON. |
| `KRISHIVA_VECTOR_STORE_PATH` | `data/vector_store` | Location of the embedded Qdrant store. |
| `KRISHIVA_COLLECTION_NAME` | `krishiva_kb` | Qdrant collection name. |
| `KRISHIVA_EMBEDDING_MODEL` | `BAAI/bge-small-en-v1.5` | Dense embedding model. Rebuild the index after changing it. |
| `KRISHIVA_RERANKER_MODEL` | `BAAI/bge-reranker-base` | Cross-encoder reranker. An empty value turns reranking off. |
| `KRISHIVA_TOP_K` | `6` | Chunks retrieved per query and passed to the answer step. |
| `KRISHIVA_USE_REAL_ENGINE` | `1` | Set to `0` to serve the stub engine. The backend also falls back to the stub when the real engine fails to build. The stub returns a fixed Rice Blast answer. |
| `KRISHIVA_LOG_LEVEL` | `INFO` | Log level. |
| `KRISHIVA_CORS_ORIGINS` | `["*"]` | JSON list of allowed origins. Set an explicit list outside local development. |

## API

### GET /health

Returns the liveness payload `{"status": "ok", "version": "0.1.0"}`.

### POST /api/v1/query

```sh
curl -s http://127.0.0.1:8000/api/v1/query \
  -H "Content-Type: application/json" \
  -d '{"query": "yellow diamond spots on my paddy leaf after foggy nights", "language": "en", "region": "Assam", "crop": "rice"}'
```

Only `query` is required. It takes 1 to 1000 characters. `language` is one of `en`, `hi`, `bn` or `as`. `region` goes into the prompt as context. `crop` narrows the search, and `region` and `crop` take up to 100 characters each.

The response has this shape. The values depend on your knowledge base.

```json
{
  "query": "yellow diamond spots on my paddy leaf after foggy nights",
  "language": "en",
  "answer": "<answer text>",
  "diagnosis": { "problem": "<problem name>", "confidence": 0.87 },
  "sources": [{ "id": "<chunk id>", "title": "<chunk title>", "score": 0.94 }],
  "request_id": "<request id>"
}
```

`diagnosis` is `null` when the top match is too weak to name a problem.

Errors use one envelope, `{"error": {"code", "message", "request_id"}}`. Stack traces never reach the client.

| Status | Code | When |
| --- | --- | --- |
| 422 | `validation_error` | The body fails validation, for example an empty query or an unsupported `language`. |
| 502 | `engine_error` | The engine failed, for example because the model server is unreachable. |

## Development

```sh
uv run pytest
uv run ruff check .
uv run ruff format .
uv run mypy .
```

The test suite has two modules, one for the query API and one for the knowledge base models. The scripts in `scripts/` check one layer each. Run one with `uv run python scripts/verify_engine.py`.

For the web client, run this from `frontend/`:

```sh
npm run lint
npm run build
```

## License

MIT. See [LICENSE](LICENSE).
