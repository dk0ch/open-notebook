# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Open Notebook is a privacy-focused, self-hosted alternative to Google's Notebook LM. It's a research assistant that processes multi-modal content (PDFs, videos, audio, web pages) and enables AI-powered chat, search, and podcast generation with support for 16+ AI providers.

**Tech Stack:**
- Backend: Python 3.11+ with FastAPI, LangChain, LangGraph
- Frontend: Next.js 15 (React) with TypeScript
- Database: SurrealDB (multi-model database)
- AI Integration: Esperanto library (multi-provider abstraction)
- Background Jobs: surreal-commands worker system
- Package Manager: `uv` (Python), `npm` (Node.js)

## Development Setup

### Initial Setup

```bash
# 1. Install dependencies
uv sync                          # Install Python dependencies
cd frontend && npm install       # Install frontend dependencies

# 2. Configure environment
cp .env.example .env
# Edit .env with your API keys (OpenAI, Anthropic, etc.)

# 3. Start all services
make start-all                   # Starts database, API, worker, and frontend
```

### Environment Requirements

- **Python:** 3.11-3.12 (managed by uv)
- **Node.js:** Latest LTS
- **Docker:** Required for SurrealDB
- **FFmpeg:** Required for video/audio processing (`brew install ffmpeg` on macOS)

## Common Commands

### Running Services

```bash
make start-all        # Start all services (recommended for development)
make stop-all         # Stop all services
make status           # Check service status

# Individual services
make database         # Start SurrealDB only
make api              # Start FastAPI backend only (uv run run_api.py)
make frontend         # Start Next.js frontend only
make worker           # Start background worker only
```

**Service URLs:**
- Frontend: http://localhost:3000
- API: http://localhost:5055
- API Docs: http://localhost:5055/docs
- SurrealDB: localhost:8000 (internal)

### Code Quality

```bash
make lint             # Run mypy type checking
make ruff             # Run ruff linter with auto-fix
make clean-cache      # Clean Python cache directories
```

### Docker Operations

```bash
make docker-push           # Build and push version tags only
make docker-push-latest    # Update v1-latest tags
make docker-release        # Full release (version + latest)
make tag                   # Create git tag from pyproject.toml version
```

## Architecture

### Four-Service Architecture

Open Notebook runs as four coordinated services:

1. **SurrealDB (port 8000)**: Multi-model database storing notebooks, sources, notes, embeddings
2. **FastAPI Backend (port 5055)**: REST API with auto-generated docs at `/docs`
3. **Background Worker**: Processes long-running tasks (transcription, embeddings, podcast generation)
4. **Next.js Frontend (port 3000)**: React-based UI

### Code Organization

```
open-notebook/
├── api/                        # FastAPI backend
│   ├── routers/               # API endpoint definitions (one per domain)
│   ├── *_service.py           # Service layer (business logic orchestration)
│   ├── models.py              # Pydantic request/response models
│   ├── auth.py                # Password authentication middleware
│   └── main.py                # FastAPI app initialization
├── open_notebook/             # Core domain logic
│   ├── domain/                # Domain models (Notebook, Source, Note, etc.)
│   │   ├── base.py           # ObjectModel base class with CRUD operations
│   │   └── *.py              # Entity models extending ObjectModel
│   ├── database/              # Data persistence layer
│   │   ├── repository.py     # Async SurrealDB operations
│   │   └── async_migrate.py  # Database migration system
│   ├── graphs/                # LangGraph AI workflows
│   │   ├── ask.py            # Question-answering pipeline
│   │   ├── chat.py           # Conversational AI
│   │   ├── source.py         # Content ingestion/processing
│   │   └── transformation.py # Content analysis workflows
│   └── utils/                 # Shared utilities
├── commands/                   # Background job definitions
│   ├── source_commands.py    # Source processing jobs
│   ├── podcast_commands.py   # Podcast generation jobs
│   └── embedding_commands.py # Embedding/re-embedding jobs
├── frontend/                   # Next.js frontend
│   ├── src/app/              # App Router pages
│   ├── src/components/       # React components
│   ├── src/lib/              # Frontend utilities, types, API clients
│   └── public/               # Static assets
├── migrations/                 # SurrealDB schema migrations (*.surrealql)
└── prompts/                   # Jinja2 prompt templates for AI workflows
```

### Key Architectural Patterns

**1. Domain-Driven Design:**
- All domain models extend `ObjectModel` (open_notebook/domain/base.py)
- Rich domain models with business logic methods
- Each model has a `table_name` class variable for SurrealDB table mapping

**2. Service Layer Pattern:**
- API routers delegate to service classes (`*_service.py`)
- Services orchestrate domain logic and coordinate between layers
- Clear separation: routers handle HTTP, services handle business logic

**3. Repository Pattern:**
- All database operations go through `open_notebook/database/repository.py`
- Async functions: `repo_create`, `repo_query`, `repo_update`, `repo_delete`
- Abstracts SurrealDB specifics from domain layer

**4. LangGraph Workflows:**
- AI processing defined as state machines in `open_notebook/graphs/`
- Multi-step workflows with state management
- Used for: content processing, chat, Q&A, transformations

**5. Background Jobs:**
- Long-running tasks processed via `commands/` using surreal-commands
- Job status tracked in SurrealDB `command` table
- Worker polls for jobs and executes asynchronously

## Working with AI Providers

Open Notebook uses the **Esperanto** library for multi-provider AI abstraction. It supports:
- **LLM:** OpenAI, Anthropic, Google (Gemini/Vertex), Mistral, Groq, Ollama, DeepSeek, xAI, OpenRouter, Azure OpenAI
- **Embeddings:** OpenAI, Google, Mistral, Ollama, Voyage, Jina
- **STT:** OpenAI (Whisper), Groq, ElevenLabs
- **TTS:** OpenAI, ElevenLabs, Google

Configuration is via environment variables in `.env`. Models are selected in the UI (Settings → Models).

### ⚠️ Known Issues with AI Providers

**Esperanto v2.7.1 Bug with Anthropic Models:**
- When using Claude models for transformations/chat, you may encounter: `anthropic.BadRequestError: 'temperature' and 'top_p' cannot both be specified`
- **Root cause:** Esperanto passes both `temperature=1.0` and `top_p=0.9` when converting Anthropic models to LangChain format, which Anthropic's API rejects
- **Workaround:** Use OpenAI GPT models instead of Claude for:
  - `default_transformation_model`
  - `large_context_model` (used when content >105k tokens)
  - `default_tools_model`
  - `default_chat_model`
- **How to fix:** Update model defaults via API or UI (Settings → Models)
- **Note:** This bug may be fixed in future Esperanto versions

### Timeout Configuration for Long Processing

**Audio/Video Transcription Timeouts:**
- Long audio files (>10 minutes) may timeout with default settings
- **Symptom:** `httpx.ReadTimeout` during Whisper API transcription
- **Fix:** Add to `.env`:
  ```bash
  ESPERANTO_LLM_TIMEOUT=600  # 10 minutes for audio/video transcription
  ```
- **When to increase:**
  - Audio files >15 minutes: 900 seconds (15 min)
  - Audio files >30 minutes: 1200 seconds (20 min)
- **Important:** Restart all services after changing timeout: `make stop-all && make start-all`

**Model Cache Behavior:**
- Model configurations are cached in running processes
- **When you change model defaults, you MUST restart services** to clear the cache
- Simply saving new defaults in the UI is not enough - old models remain in memory
- Always run `make stop-all && make start-all` after model configuration changes

## Database Migrations

Migrations are SurrealQL files in `migrations/` directory, numbered sequentially (1.surrealql, 2.surrealql, etc.).

**Creating a new migration:**
1. Create `migrations/N.surrealql` (where N = next number)
2. Write SurrealQL DDL statements
3. Restart API - migrations run automatically on startup via `AsyncMigrationManager`

**Current version tracked in:** `database_version` table

## Content Processing Pipeline

When a source is uploaded:

1. **File Upload** → Saved to `data/uploads/`
2. **Source Command** → Background job created in `commands/source_commands.py`
3. **Content Extraction** → Uses content-core library with engine selection (Docling, PyMuPDF, BeautifulSoup, Firecrawl)
4. **Video/Audio** → FFmpeg extracts audio → Whisper transcribes (requires FFmpeg installed)
5. **Transformations** → Optional AI analysis (summary, key points, custom prompts)
6. **Embeddings** → Vector embeddings created for semantic search (if enabled)
7. **Storage** → Text content and metadata saved to SurrealDB

## Frontend-Backend Communication

- Frontend uses `fetch` to call REST API at `/api/*` endpoints
- Runtime config loaded from `/api/runtime-config` (sets `API_URL` for deployments)
- Authentication: Bearer token in `Authorization` header (if `OPEN_NOTEBOOK_PASSWORD` set)
- CORS enabled for local development (localhost:3000 ↔ localhost:5055)

### Next.js API Proxy Configuration

**Important:** Next.js requires explicit rewrites to proxy API requests to the FastAPI backend in development.

The configuration in `frontend/next.config.ts` should include:

```typescript
async rewrites() {
  return [
    {
      source: '/api/:path*',
      destination: process.env.API_URL || 'http://localhost:5055/api/:path*',
    },
  ]
},
```

**Without this configuration:**
- Frontend API calls to `/api/*` will return 404 errors
- Chat, source processing, and all API interactions will fail
- Direct API calls to `localhost:5055` will work, but frontend calls to `localhost:3000/api/*` won't

**Verification:**
Test the proxy is working with: `curl http://localhost:3000/api/models`

**Note:** This is only required for local development. In production Docker, the API runs on the same domain so no proxy is needed.

## Important File Locations

- **API entry point:** `run_api.py`
- **Frontend entry point:** `frontend/src/app/layout.tsx`
- **Environment config:** `.env` (never commit this!)
- **Docker compose:** `docker-compose.yml`, `docker-compose.dev.yml`, `docker-compose.single.yml`
- **Dependencies:** `pyproject.toml` (Python), `frontend/package.json` (Node.js)
- **Upload storage:** `data/uploads/` (temporary files)
- **Database storage:** `surreal_data/` (SurrealDB data files)

## Port Configuration

- **5055:** FastAPI backend (required)
- **3000:** Next.js frontend dev server (8502 in production Docker)
- **8000:** SurrealDB (internal to Docker network)

For remote deployments, set `API_URL` environment variable to match how users access the frontend.

## Debugging

**Check service status:**
```bash
make status
```

**View logs:**
- API logs: console output from `make api`
- Worker logs: console output from `make worker`
- Database logs: `docker logs open-notebook-surrealdb-1`
- Frontend logs: console output from `make frontend`

**Enable debug logging:**
Set environment variables in `.env`:
```bash
LANGCHAIN_TRACING_V2=true      # Enable LangSmith tracing
LANGCHAIN_API_KEY=<your-key>   # For detailed AI workflow debugging
```

## Testing

Currently limited test coverage. Tests located in `tests/`:
- `test_models_api.py` - Model API endpoint tests
- `test_source_chat*.py` - Source chat functionality tests

Run with: `pytest` (after installing dev dependencies: `uv sync --dev`)

## Version Management

Version is defined in `pyproject.toml`. Update it there, then:
```bash
make tag                # Creates and pushes git tag
make docker-release     # Builds and pushes Docker images
```

## Frontend Development Notes

- **App Router:** Next.js 15 with App Router (not Pages Router)
- **Styling:** Tailwind CSS with shadcn/ui components
- **State:** React hooks (no global state management currently)
- **API Client:** Custom fetch wrappers in `frontend/src/lib/`
- **Hot Reload:** Works out of the box with `make start-all`

## Video/Audio Processing

Requires **FFmpeg** to be installed on the system:
```bash
brew install ffmpeg  # macOS
```

Without FFmpeg, video uploads will fail with "extract_best_audio_from_video" error.

## Accepted File Types

Update file accept list in: `frontend/src/components/sources/steps/SourceTypeStep.tsx`

Currently supports: PDF, DOC(X), TXT, MD, EPUB, MP4, MP3, WAV, M4A, PPT(X), XLS(X), Images (JPG/PNG/TIFF), Archives (ZIP/TAR/GZ)
