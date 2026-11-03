# llm-observability-platform

A small, self-hostable system for inspecting the internal behavior of LLM
applications. It captures every request, agent step, and tool call as
structured trace data with cost and latency, and presents it in a dashboard.

## Component layout

The repository is organized into three application components plus supporting
assets and documentation:

| Path | Component | Description |
|---|---|---|
| `backend/` | Backend | FastAPI + SQLAlchemy service: ingestion endpoint, query API, authentication, cost service, and the Admin CLI. Python 3.12. |
| `sdk/observability/` | SDK | Python 3.12 library that instruments host application code (`@observe`, `wrap_llm`) and transmits spans asynchronously. |
| `dashboard/` | Dashboard | React + TypeScript (Vite) single-page app: API-key gate, trace list with filters and pagination, and the trace-detail span tree. |
| `docker/` | Deployment | Docker Compose assets wiring Backend, Database (PostgreSQL), and Dashboard for local self-hosting. |
| `docs/adr/` | Docs | Architecture Decision Records: a template (`0000-template.md`) and the numbered ADRs recording key design decisions. |

The Python packages (`backend/`, `sdk/`) each declare their dependencies and
shared pytest configuration in a `pyproject.toml`. The dashboard uses Vitest,
React Testing Library, and fast-check for testing (see `dashboard/package.json`).

The feature specification — requirements, design, and the implementation plan —
lives under `.kiro/specs/llm-observability-platform/`.
