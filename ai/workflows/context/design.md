# Design Context

Artifacts live under [design/](../../../design/).

## Structure

- `architecture.md` — cross-cutting system architecture; read by all personalities at this phase
- `data-model.md` — cross-cutting; informs engine, db, and controller subsystems
- `engine/` — computation algorithms and interfaces
- `db/` — persistence schema and access patterns
- `controller/` — API surface and pipeline orchestration
- `ux/` — information architecture, component specs, visual design

## What Design Is

Design defines the architecture, data models, algorithms, and interfaces that satisfy the requirements.

- Component structure and relationships
- Data models and schemas
- Algorithms and formulas
- Interfaces and contracts between components
- Technology-specific but implementation-independent — multiple implementations could realize the same design

Examples:
- "Timeline data stored as JSON files with one file per scenario"
- "The `TimelineYear` data structure contains: year, ages dict, phase flags, and cumulative inflation factors"
- "RMD calculation uses the formula: `prior_year_balance / uniform_lifetime_divisor[age]`"
- "The Assumptions Engine exposes a `get_return(asset_class, year) -> float` interface"

## What Design Excludes

- **No user needs or business justifications** — "users need to see their RMD amounts" is a requirement
- **No specific language or library choices** — "use Python's `dataclasses` module" is implementation
- **No coding conventions or formatting** — "JSON with 2-space indentation" is implementation
- **No function signatures or class definitions** — exact `fn foo(x: i32) -> bool` is implementation
- **No framework-specific patterns** — React hooks, Tauri commands, etc. are implementation

Test: could this be implemented in a different language without changing the design? If not, it's leaking implementation.

## Altitude

- **Forest:** read `architecture.md`, skim `data-model.md`, then survey each subsystem directory at breadth
- **Branch:** read cross-cutting docs relevant to your domain deeply; read your domain's subtree deeply
- **Leaf:** read `architecture.md` for orientation, then read your subsystem directory exhaustively
