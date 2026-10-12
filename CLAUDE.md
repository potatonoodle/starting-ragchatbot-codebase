# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- Always use `uv` to run the server and any Python — never use `pip` directly. Add deps with `uv add <pkg>`; run Python files with `uv run python <file>.py` (never bare `python`).
- Start the server **from `backend/`** (paths like `../docs`, `../frontend`, `./chroma_db` are relative to it):
  `cd backend && uv run uvicorn app:app --reload --port 8000` (or `./run.sh` in Git Bash)
- Requires `ANTHROPIC_API_KEY` in a root `.env` (copy `.env.example`).
- There are no tests, linter, or formatter configured. Verify changes by running the server and hitting `http://localhost:8000/api/courses` and `POST /api/query` (`{"query": "...", "session_id": null}`).

## Architecture

FastAPI (`backend/app.py`) serves both the API and the static `frontend/` (vanilla JS, no build step). `RAGSystem` (`rag_system.py`) wires everything together.

Query flow: `POST /api/query` → `RAGSystem.query` → `AIGenerator` makes Claude call #1 with the `search_course_content` tool → if Claude calls it, `ToolManager` → `CourseSearchTool` → `VectorStore.search` → results go back in Claude call #2 **with tools removed** (so at most one search per query). Sources are read from `CourseSearchTool.last_sources` afterwards and then reset.

`VectorStore` uses two ChromaDB collections:
- `course_catalog` — one doc per course; used to fuzzy-resolve a course name to its exact title. Lessons stored as a JSON string in metadata (`lessons_json`).
- `course_content` — text chunks with `course_title` / `lesson_number` / `chunk_index` metadata, used for filtered semantic search.

New tools: subclass `Tool` in `search_tools.py` and register with `ToolManager`; `ToolManager.get_last_sources` picks up any tool with a `last_sources` attribute.

## Gotchas

- **Course title is the unique ID** (catalog ID and chunk ID prefix). Startup skips any course whose title is already in ChromaDB, so after changing `document_processor.py`, chunking config, or editing an existing file in `docs/`, delete `backend/chroma_db/` and restart to re-index.
- Course files must follow the format parsed in `DocumentProcessor.process_course_document`: `Course Title:` / `Course Link:` / `Course Instructor:` header, then `Lesson N: <title>` sections each optionally followed by a `Lesson Link:` line. Only plain text is actually read, despite `.pdf`/`.docx` being accepted.
- Sessions are in-memory only (lost on restart). History is injected into the system prompt as text, not as API messages. `MAX_HISTORY` counts exchanges (kept as `MAX_HISTORY * 2` messages).
- Model and RAG settings (chunk size, overlap, max results) live in `backend/config.py`.
