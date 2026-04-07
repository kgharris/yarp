# Swarm Definition: Technology Stack Conflicts

## Objective

Find language, assumptions, or references that conflict with the decided
technology stack: **Rust / Tauri / React + TypeScript + shadcn/ui**.

## Scope

All `.md` files under `requirements/`, `design/`, `implementation/`, and
`tests/`. Exclude `personalities/` and `reviews/`.

## Search Criteria

A finding is anything that assumes, implies, or names a technology not in the
decided stack.

### Direct technology references
- Python, pip, PyYAML, dataclasses, `@dataclass`, `json.dump`, pytest, mypy,
  virtualenv, poetry, conda, or any Python library name.
- Flask, Django, FastAPI, Express, or any HTTP server framework.
- HTTP routes, REST endpoints, `GET /api/...`, `POST /api/...`, or any
  language implying an HTTP server between Controller and UX.
- SQLite, PostgreSQL, or any database engine (the persistence layer uses
  JSON-on-disk via serde_json, not a database server).
- Electron, Qt, GTK, or any desktop framework other than Tauri.
- Any CSS framework other than shadcn/ui / Tailwind (e.g., Bootstrap,
  Material UI).

### Implicit technology assumptions
- References to "the server" or "server-side" when describing the
  Controller — in Tauri the Controller runs in-process, not as a server.
- References to "API endpoints" or "HTTP API" when describing Controller
  commands — the correct framing is Tauri commands invoked via `invoke()`.
- References to "the browser" as a deployment target — the UX runs inside
  a Tauri WebView, not a standalone browser.
- References to "pip install", "npm start" as a dev server, or any
  deployment workflow that assumes a client-server HTTP split.
- Language like "send a request to the backend" — in Tauri the call is
  an IPC invoke, not an HTTP request.

### Language that should be updated
- "Python dataclass" → should reference Rust struct.
- "JSON files formatted with `json.dump`" → should reference `serde_json`.
- "Parse YAML with PyYAML" → should reference a Rust YAML crate or note
  that YAML is not used.
- "Use Recharts library" → acceptable (Recharts is a React library), but
  flag if context implies a non-React charting solution.
- Any mention of `class` hierarchies without noting the Rust enum/trait
  equivalent (design docs should use Rust idiom vocabulary).

## Context for Agents

The decided stack (from [architecture.md](../design/architecture.md)):

| Layer | Technology | Rationale |
|---|---|---|
| Engine (Model) | Rust (library crate) | Pure computation, no Tauri dependency; portable to WASM or CLI |
| DB (Model — persistence) | Rust (library crate) | JSON read/write via serde_json |
| Controller | Tauri commands (`#[tauri::command]`) | Typed async RPC boundary between engine and UX |
| UX (View) | React + TypeScript + shadcn/ui | Component library, forms, charts |
| Desktop shell | Tauri | Native Windows/Mac/Linux installer; WebView2 frontend |

Key constraints:
- The Engine crate has zero Tauri dependency.
- The UX communicates with the Controller exclusively via `invoke()` — no
  direct filesystem or engine access.
- "API" in this project means Tauri commands, not HTTP endpoints.
- "Server" and "client" are not the right mental model — both sides run
  on the same machine in the same process tree.