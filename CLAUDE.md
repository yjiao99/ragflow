# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RAGFlow is an open-source RAG (Retrieval-Augmented Generation) engine based on deep document understanding. It's a full-stack application with:

- Python backend (Quart-based async API server — Quart is the async reimplementation of Flask)
- React/TypeScript frontend (built with vitejs)
- Background task executor workers (separate Python processes, Redis-queue-driven)
- Peewee ORM for database models (not SQLAlchemy)
- Multiple data stores (MySQL/PostgreSQL, Elasticsearch/Infinity/OpenSearch/OceanBase, Redis, MinIO)

## Architecture

### Runtime Architecture

RAGFlow runs as **two separate Python process types**, orchestrated by `docker/launch_backend_service.sh`:

- **API Server** (`api/ragflow_server.py`): Quart-based async HTTP server handling HTTP requests
- **Task Executors** (`rag/svr/task_executor.py`): Background workers processing documents from Redis streams. Multiple instances run in parallel (controlled by `WS` env var). Each consumes from priority-ordered Redis streams (`te.1.common`, `te.0.common`), using consumer groups for load distribution.

Key consequence: task executors import a different code surface than the API server, so always check which process a module is meant for.

### Backend API (`/api/`)

- **App factory**: `api/apps/__init__.py` — creates the Quart app, configures auth (`login_required` decorator, JWT + API token + session fallback), and dynamically discovers/registers blueprints
- **Three API patterns**:
  - **RESTful APIs** in `api/apps/restful_apis/` — newer pattern with Pydantic request validation, service layer in `api/apps/services/`, routes registered under `/api/v1`
  - **Legacy APIs** in `api/apps/*_app.py` — older pattern using `@validate_request()`, routes registered under `/v1/<page_name>`
  - **SDK APIs** in `api/apps/sdk/` — registered under `/v1/`
- **Route registration**: Automatically discovered from files in `api/apps/restful_apis/`, `api/apps/sdk/`, and `api/apps/*_app.py`. Each file creates a Blueprint named `manager` and defines routes using `@manager.route()`
- **Services**: `api/db/services/` — business logic wrapping Peewee model operations. `api/apps/services/` — service layer for the RESTful APIs
- **Models**: `api/db/db_models.py` — Peewee ORM models with pooled MySQL/PostgreSQL connections, custom `JSONField`/`ListField` types, retry logic on connection loss

### Core Processing (`/rag/`)

- **Document ingestion pipeline**: `rag/flow/pipeline.py` — `Pipeline` (extends `agent.canvas.Graph`) orchestrates the ingestion DAG. Components: File (fetches binary from storage), Parser (dispatches to `deepdoc.parser` based on file type), TokenChunker/TitleChunker (splits into chunks), Tokenizer (computes full-text tokens + embedding vectors), Extractor (LLM-based extraction). Data flows via Pydantic `*FromUpstream` schemas.
- **Document parsing**: `deepdoc/` — PDF parsing (vision-based OCR, layout analysis, table structure recognition) and format-specific parsers (DOCX, XLSX, PPT, Markdown, HTML, images). All parsers normalize to a common structure (list of bbox dicts for PDFs, `{text, doc_type_kwd}` for others).
- **LLM Integration**: `rag/llm/` — factory pattern with runtime class discovery. `chat_model.py` (30+ providers via OpenAI SDK and LiteLLM wrappers), `embedding_model.py`, `rerank_model.py`, `cv_model.py` (image-to-text), `sequence2txt_model.py` (ASR), `tts_model.py`. Use `LLMBundle` (from `api.db.services.llm_service`) as the unified interface.
- **Graph RAG**: `rag/graphrag/` — multi-phase pipeline: per-document subgraph extraction (LLM or spaCy NER), Leiden community detection, entity resolution, community summarization. Entities/relations/reports are indexed as chunks alongside regular text chunks, differentiated by `knowledge_graph_kwd`.
- **Search**: `rag/nlp/search.py` — `Dealer` class combines vector similarity + BM25 + re-ranking. `KGSearch` extends it for graph-aware retrieval (entity resolution, n-hop enrichment).

### Agent System (`/agent/`)

- **Execution engine**: `agent/canvas.py` — `Canvas` (extends `Graph`) executes the DAG. Components are run in topological order via `_run_batch`, each receiving upstream outputs as kwargs. Control-flow components (`Categorize`, `Switch`, `Iteration`, `Loop`) dynamically modify the execution path.
- **Component base**: `agent/component/base.py` — `ComponentBase` with `invoke(**kwargs)` / `invoke_async(**kwargs)` lifecycle. Variable references (`{component_id@output_var}` or `{sys.query}`) are resolved from the canvas graph at runtime.
- **Components**: Modular workflow components in `agent/component/` — Begin, LLM, Agent (tool-calling LLM), Categorize, Switch, Iteration, Loop, Message, Invoke (HTTP), and data manipulation nodes. Auto-discovered by `__init__.py`.
- **Templates**: Pre-built agent workflows as JSON DSL files in `agent/templates/`. Each contains a complete `components` DAG, `path`, and `globals`.
- **Tools**: `agent/tools/` — Retrieval, web search (DuckDuckGo, Google, Tavily, SearXNG), academic search (ArXiv, PubMed, Google Scholar, Wikipedia), code execution, SQL execution, email, GitHub, finance data, translation, weather. Tools implement `ToolBase` (extends `ComponentBase`) and produce OpenAI-compatible function descriptors.
- **Plugins**: `agent/plugin/` — plugin system using `pluginlib` for loading external LLM tool plugins from `embedded_plugins/`.

### Frontend (`/web/`)

- React/TypeScript with vitejs framework
- shadcn/ui components (Radix UI primitives + Tailwind CSS)
- `@tanstack/react-query` for server state (cache keys, mutations, invalidation)
- Zustand for local state (primarily agent canvas graph store)
- `react-router` v7 with lazy-loaded pages
- `react-i18next` for i18n (17 languages)
- Axios for HTTP with a layered pattern: endpoint definitions (`utils/api.ts`) → HTTP client (`utils/next-request.ts`) → service layer (`services/`) → query hooks (`hooks/use-*-request.ts`) → components
- `@xyflow/react` for the agent workflow canvas
- `react-hook-form` + `zod` for form validation
- Two API proxy prefixes: `webAPI = '/v1'` (legacy) and `restAPIv1 = '/api/v1'` (RESTful)

## Common Development Commands

### Backend Development

```bash
# Install Python dependencies
uv sync --python 3.13 --all-extras
uv run python3 download_deps.py
pre-commit install

# Start dependent services (MySQL, Redis, Elasticsearch/Infinity, MinIO)
docker compose -f docker/docker-compose-base.yml up -d

# Run backend (requires services to be running)
source .venv/bin/activate
export PYTHONPATH=$(pwd)
bash docker/launch_backend_service.sh

# Run tests
uv run pytest                                    # Run all tests
uv run pytest -m p1                             # Run priority 1 tests only
uv run pytest -m p2                             # Run priority 2 tests only
uv run pytest path/to/test_file.py              # Run specific test file
uv run pytest path/to/test_file.py::test_func   # Run specific test function

# Linting and formatting
ruff check
ruff format
```

### Frontend Development

```bash
cd web
npm install
npm run dev        # Development server
npm run build      # Production build
npm run lint       # ESLint
npm run test       # Jest tests
```

### Docker Development

```bash
# Full stack with Docker
cd docker
docker compose -f docker-compose.yml up -d

# Check server status
docker logs -f ragflow-server

# Rebuild images
docker build --platform linux/amd64 -f Dockerfile -t infiniflow/ragflow:nightly .
```

## Key Configuration Files

- `docker/.env` - Environment variables for Docker deployment
- `docker/service_conf.yaml.template` - Backend service configuration
- `pyproject.toml` - Python dependencies and project configuration
- `web/package.json` - Frontend dependencies and scripts

## Testing

- **Python**: pytest with markers (p1/p2/p3 priority levels)
- **Frontend**: Jest with React Testing Library
- **API Tests**: HTTP API and SDK tests in `test/` and `sdk/python/test/`

## Database Engines

RAGFlow supports switching between Elasticsearch (default) and Infinity:

- Set `DOC_ENGINE=infinity` in `docker/.env` to use Infinity
- Requires container restart: `docker compose down -v && docker compose up -d`

## Query Flow Architecture

When a user sends a chat message, the flow is:

1. **Frontend**: React Query hook (`use-chat-request.ts`) → Service layer (`next-chat-service.ts`) → Axios client (`next-request.ts`) → `POST /api/v1/chat/completions`
2. **Backend Route**: `api/apps/restful_apis/chat_api.py::session_completion()` handles auth, loads chat/session, calls `async_chat()`
3. **Chat Orchestration**: `api/db/services/dialog_service.py::async_chat()`:
   - Loads models via `LLMBundle` (embedding, LLM, rerank, TTS)
   - Performs retrieval via `rag/nlp/search.py::Dealer` (vector + BM25 + rerank)
   - Constructs prompt with retrieved knowledge
   - Streams response via `LLMBundle.async_chat_streamly_delta()`
4. **Response**: SSE stream back to frontend with answer chunks and references

## Frontend Conventions

### CSS and Layout Debugging
When fixing CSS/layout issues (especially flex truncation, ellipsis, or element sizing), **always inspect the full parent hierarchy** for `flex-shrink`, `min-width`, and `overflow` constraints before applying fixes like `min-w-0`.

### Query Key Factory (Mandatory)
**Never write raw `queryKey` arrays inline.** Always use a query key factory object that returns `as const` tuples:

```ts
// ❌ Bad — raw array
queryClient.invalidateQueries({
  queryKey: [LLMApiAction.AddedProviders, params.provider_name],
});

// ✅ Good — factory reference
queryClient.invalidateQueries({
  queryKey: LlmKeys.instanceModels(params.provider_name, params.instance_name),
});
```

### Network Request Layering
HTTP requests are organized in three layers. **Never import `@/utils/request`, `@/utils/next-request`, or `@/utils/api` directly inside a hook**:
1. `src/hooks/use-xx-request.ts(x)` — React Query hooks; only call the service layer
2. `src/services/xx-service.ts` — Register endpoints via `registerNextServer`
3. `src/utils/next-request.ts` — The single axios instance

### Shared UI Component Lock
The folder `src/components/ui/` is the project's **shared UI library**. **Do not modify** any file under it. If a component doesn't meet requirements, **wrap or compose it** in a new component outside this folder.

### i18n Style
For `en.ts`: Use sentence case — first word capitalized, rest lowercase (e.g., `referenceAnswer: 'Reference answer'`). Proper nouns remain as-is.

## Development Environment Requirements

- Python 3.10-3.13
- Node.js >=18.20.4
- Docker & Docker Compose
- uv package manager
- 16GB+ RAM, 50GB+ disk space

1. Think before acting. Read existing files before writing code.
2. Be concise in output but thorough in reasoning.
3. Prefer editing over rewriting whole files.
4. Do not re-read files you have already read.
5. Test your code before declaring done.
6. No sycophantic openers or closing fluff.
7. Keep solutions simple and direct.
8. User instructions always override this file.
