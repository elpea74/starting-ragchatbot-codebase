# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## How to Run

**Prerequisites:** Python 3.13, `uv` package manager, Anthropic API key.

**Dependency management:** All dependencies are managed through `uv` and declared in `pyproject.toml`. Always use `uv` — never use `pip` directly. This includes installing new packages (`uv add <package>`), syncing (`uv sync`), and running Python commands (`uv run ...`).

```bash
# 1. Install dependencies
uv sync

# 2. Set up environment — create .env in project root
echo "ANTHROPIC_API_KEY=your_key_here" > .env

# 3. Start the server
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app serves both frontend and API at `http://localhost:8000`. API docs at `/docs`.

On startup, the app loads all `docs/*.txt` files into ChromaDB automatically.

## Dependency Management

**Always use `uv`. Never use `pip` directly.**

```bash
# Install/sync all dependencies
uv sync

# Add a new package
uv add <package>

# Run any Python command
uv run <command>
```

Dependencies are declared in `pyproject.toml` and locked in `uv.lock`.

## Architecture

**Monorepo layout:** `frontend/` (static HTML/CSS/JS), `backend/` (Python FastAPI), `docs/` (course text files).

**Request flow:**
1. `frontend/script.js` captures user input → `POST /api/query`
2. `backend/app.py` (FastAPI) validates request, delegates to `RAGSystem`
3. `backend/rag_system.py` orchestrates: SessionManager → AIGenerator → ToolManager → VectorStore
4. `backend/ai_generator.py` calls Anthropic Claude with tool definitions; Claude decides whether to call `search_course_content`
5. `backend/search_tools.py` executes search via `backend/vector_store.py` (ChromaDB with sentence-transformers embeddings)
6. Response flows back through the chain with source citations

**Key detail — tool-based RAG:** Not a fixed retrieve-then-generate pipeline. Claude uses Anthropic's `tool_choice: "auto"` to decide per-query whether a vector search is needed. Course-specific questions trigger search; general questions get direct answers.

**Key files:**
- `backend/config.py` — all settings (model name, chunk size 800, overlap 100, max results 5, max history 2)
- `backend/models.py` — Pydantic models: `Course`, `Lesson`, `CourseChunk`
- `backend/document_processor.py` — parses course `.txt` files (expected format: "Course Title:", "Course Link:", "Course Instructor:" header lines, then "Lesson N: Title" sections)
- `backend/session_manager.py` — in-memory session storage (no persistence across restarts)

**Vector storage:** Two ChromaDB collections in `backend/chroma_db/`: `course_catalog` (course metadata, title as ID) and `course_content` (lesson chunks with embeddings via `all-MiniLM-L6-v2`).
