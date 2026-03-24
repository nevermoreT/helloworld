# AGENTS.md - DeerFlow Coding Agent Guide

This file provides essential information for agentic coding agents operating in this repository.

## Repository Overview

DeerFlow is a full-stack "super agent harness" with:
- **Backend**: Python 3.12, LangGraph + FastAPI gateway, sandbox/tool system, memory, MCP integration
- **Frontend**: Next.js 16 + React 19 + TypeScript + pnpm
- **Local dev entrypoint**: `make dev` starts all services on `http://localhost:2026`

## Build/Lint/Test Commands

### Prerequisites Check
```bash
make check              # Verify Node.js 22+, pnpm, uv, nginx installed
make install            # Install all dependencies (frontend + backend)
```

### Backend (from `backend/` directory)

```bash
make lint               # Lint with ruff
make test               # Run all tests (pytest)
make format             # Format code with ruff

# Run a single test file
PYTHONPATH=. uv run pytest tests/test_client.py -v

# Run a single test function
PYTHONPATH=. uv run pytest tests/test_client.py::TestClientInit::test_default_params -v

# Run tests matching a pattern
PYTHONPATH=. uv run pytest tests/ -k "memory" -v
```

### Frontend (from `frontend/` directory)

```bash
pnpm lint               # Lint with ESLint
pnpm lint:fix           # Auto-fix lint issues
pnpm typecheck          # TypeScript type check

# Build (requires BETTER_AUTH_SECRET)
BETTER_AUTH_SECRET=local-dev-secret pnpm build
```

### Full Application (from root directory)

```bash
make dev                # Start all services (LangGraph + Gateway + Frontend + Nginx)
make stop               # Stop all services
```

## Code Style Guidelines

### Backend (Python)

**Imports** (from `backend/ruff.toml`):
- Line length: 240 characters
- Python 3.12+ with type hints
- Double quotes, space indentation
- Import order enforced by ruff isort: `deerflow`, `app` are first-party packages

```python
# Standard library
import sys
from pathlib import Path
from typing import Annotated, NotRequired, TypedDict

# Third-party
from langchain.agents import AgentState
from fastapi import FastAPI

# First-party (deerflow, app)
from deerflow.agents import make_lead_agent
from app.gateway.app import app
```

**Naming Conventions**:
- `snake_case` for functions, variables, modules
- `PascalCase` for classes, TypedDicts
- `UPPER_SNAKE_CASE` for constants
- Private functions: prefix with `_`
- Middleware classes: `*Middleware` suffix

**Type Hints**:
- Use `TypedDict` for structured dictionaries
- Use `NotRequired` for optional TypedDict fields
- Use `Annotated` with custom reducers for state fields
- Prefer `list[str]` over `List[str]` (modern Python syntax)
- Use `str | None` instead of `Optional[str]`

**Error Handling**:
- Raise exceptions with descriptive messages
- Use try/except for external API calls
- Log errors appropriately before re-raising

### Frontend (TypeScript/React)

**Imports** (from `frontend/eslint.config.js`):
- Use `import type` for type-only imports (enforced)
- Import order: builtin → external → internal → parent → sibling → index
- `@/*` aliases for internal imports
- CSS imports in separate group

```typescript
import type { AIMessage } from "@langchain/langgraph-sdk";

import { useMutation, useQuery } from "@tanstack/react-query";
import { useCallback, useEffect } from "react";

import type { LocalSettings } from "@/core/settings";

import { getAPIClient } from "@/core/api";
import "./styles.css";
```

**Naming Conventions**:
- `camelCase` for functions, variables
- `PascalCase` for components, types, interfaces
- `UPPER_SNAKE_CASE` for constants
- Hook functions: `use*` prefix
- Type aliases: `*Options`, `*Props`, `*State` suffixes

**TypeScript Config**:
- Strict mode enabled
- `noUncheckedIndexedAccess: true`
- `verbatimModuleSyntax: true`
- Path alias: `@/*` maps to `./src/*`

## Project Structure

```
deer-flow/
├── Makefile                    # Root commands
├── config.yaml                 # Main application configuration
├── extensions_config.json      # MCP servers and skills config
├── backend/
│   ├── Makefile               # Backend commands
│   ├── pyproject.toml         # Python dependencies
│   ├── ruff.toml              # Lint/format config
│   ├── packages/harness/      # deerflow-harness package
│   │   └── deerflow/          # Core agent framework (import: deerflow.*)
│   ├── app/                   # Application layer (import: app.*)
│   │   ├── gateway/           # FastAPI Gateway API
│   │   └── channels/          # IM platform integrations
│   └── tests/                 # Test suite
├── frontend/
│   ├── package.json           # Node dependencies
│   ├── eslint.config.js       # ESLint config
│   ├── tsconfig.json          # TypeScript config
│   └── src/
│       ├── app/               # Next.js App Router
│       ├── components/        # React components
│       └── core/              # Core business logic
└── skills/                    # Agent skills
    ├── public/                # Built-in skills
    └── custom/                # Custom skills (gitignored)
```

## Critical Rules

### Harness/App Boundary (Backend)
- **App imports deerflow** - ALLOWED
- **deerflow imports app** - FORBIDDEN (enforced by `test_harness_boundary.py` in CI)

```python
# ALLOWED
from deerflow.config import get_app_config

# FORBIDDEN (will fail CI)
from app.gateway.routers.uploads import ...
```

### Test-Driven Development
- Every new feature or bug fix MUST have unit tests
- Tests in `backend/tests/` with naming `test_<feature>.py`
- Run `make test` before and after changes

### Documentation Updates
- Update `README.md` for user-facing changes
- Update `backend/CLAUDE.md` for development changes
- Keep documentation synchronized with code

## Pre-Commit Checklist

Before submitting changes:

1. **Backend**: `cd backend && make lint && make test`
2. **Frontend**: `cd frontend && pnpm lint && pnpm typecheck`
3. **Frontend build** (if UI/auth changes): `BETTER_AUTH_SECRET=... pnpm build`

## Common Gotchas

- **Proxy env vars** can break `pnpm install` - unset if needed
- **BETTER_AUTH_SECRET** is required for frontend production build
- **`make config`** aborts if `config.yaml` already exists (by design)
- **`make dev`** includes cleanup; interruption noise is expected
- **`pnpm check`** is broken - use `pnpm lint && pnpm typecheck` instead

## Key Files Reference

| File | Purpose |
|------|---------|
| `backend/CLAUDE.md` | Backend architecture and design patterns |
| `backend/AGENTS.md` | Points to CLAUDE.md |
| `frontend/AGENTS.md` | Frontend agent architecture |
| `.github/copilot-instructions.md` | Copilot onboarding guide |
| `.github/workflows/backend-unit-tests.yml` | CI workflow |
