# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WaniKani MCP server — an MCP (Model Context Protocol) server that connects WaniKani (Japanese learning platform) to AI assistants. It syncs WaniKani data to a local database and exposes tools/resources over MCP stdio protocol.

## Commands

All commands use [Task](https://taskfile.dev/) runner with `uv` as the Python package manager.

```bash
task dev              # Run stdio MCP server (for Claude Code integration)
task test             # Run pytest
task lint             # Run ruff check + format (auto-fix)
task lint-check       # Run ruff check only (no fix)
task type-check       # Run type checker (uvx ty check)
task check            # All checks: lint-check, type-check, test
task db-migrate       # Apply Alembic migrations
task db-create-migration -- "description"  # Create new migration
task install          # uv sync
```

Run a single test:
```bash
uv run pytest tests/test_models.py -k "test_name"
```

## Architecture

**Server startup flow**: `__main__.py` -> `server.py` (ServerManager) -> starts `sync_service` (APScheduler background sync every 30 min) + `mcp_server.py` (stdio MCP protocol).

**Key constraint**: Server runs in stdio mode — all logging goes to `server.log` file, never stdout/stderr, to avoid corrupting MCP protocol communication.

**MCP tools** (in `mcp_server.py`): `register_user`, `get_status`, `get_leeches`, `sync_data` — each requires an `mcp_api_key` param (except `register_user` which takes a `wanikani_api_key`).

**MCP resources**: `wanikani://user_progress`, `wanikani://review_forecast`, `wanikani://item_database` — accessed with `?mcp_api_key=` query param.

**Auth flow**: User provides WaniKani API key -> validated against WaniKani API -> server generates MCP API key (URL-safe 32-byte token) -> stored in DB -> all subsequent calls use MCP key.

**Data sync** (`sync_service.py`): APScheduler runs `_sync_user_data` for users who haven't synced in >1 hour. Uses semaphore (max 3 concurrent syncs). Initial sync is full; subsequent syncs use `updated_after` for incremental updates.

**WaniKani client** (`wanikani_client.py`): Async httpx client with automatic pagination and rate limiting (60 req/min shared across instances).

**Database**: SQLModel ORM. Default SQLite, supports PostgreSQL. Models mirror WaniKani API entities: User, Subject, Assignment, Review, ReviewStatistic, LevelProgression, StudyMaterial, SyncLog. Migrations via Alembic.

**Config** (`config.py`): pydantic-settings reading from environment. Key vars: `DATABASE_URL`, `SYNC_INTERVAL_MINUTES`, `MAX_CONCURRENT_SYNCS`, `LOG_LEVEL`, `SENTRY_DSN`.

## Code Style

- Python 3.12+, fully async
- Ruff for linting/formatting (line length 88, rules: E/W/F/I/N/UP/B/C4/SIM)
- SQLModel for ORM (Pydantic + SQLAlchemy)
- JSON fields use `sa_column=Column(JSON)` pattern in models
