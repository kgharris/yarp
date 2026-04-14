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

## Principles vs. Requirements Scope

Some documents in this project contain **forward-looking guidance** — they name responsibilities, reserve architectural territory, and introduce domain concepts that are not yet scoped for MVP. These documents must not generate findings solely because a named concept or responsibility has no current design coverage or MVP-tagged requirement:

- `principles.md` files (bedrock, phase-level, and subsystem-level)
- `requirements/conceptual-model.md`
- `design/architecture.md`

The correct reading of an undesigned concept in these documents is: the architecture has reserved that space for the future. It is not a gap.

**Actual design files are not exempt.** Design coverage in `design/data-model.md`, `design/engine/`, `design/db/`, `design/controller/`, and `design/ux/` must be anchored to an MVP-tagged requirement. Design that exists without a corresponding `MVP`-tagged `R:` row is unanchored — that is a legitimate finding.
