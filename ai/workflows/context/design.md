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

## Scope Gate — Required Before Every Finding

**Before raising any finding, consult the requirements.** This check is mandatory — it is not optional and it is not implied by domain knowledge or bedrock principles alone.

**Finding about missing design coverage** — verify the corresponding `R:` row exists and is tagged `MVP`:
- Tagged `FUT` → do not raise the finding. Absent design for a `FUT` requirement is intentional.
- Row does not exist → do not raise the finding. Unspecified requirements have no design obligation.
- Tagged `MVP` → the finding is in scope; proceed.

**Finding about existing design behavior** — verify no MVP requirement already specifies that behavior:
- If an MVP-tagged `R:` row already resolves the question the finding is raising, the finding is invalid.
- Requirements are the authoritative answer to "what should this do." Bedrock principles establish the framework; requirements instantiate it. Check both before concluding something is ambiguous.

**Domain expertise is not a scope override.** A personality file may list domain areas to watch (SS, RMD, CPI-W, IRMAA, etc.). These represent expertise to apply when examining in-scope features — they are not a scope-independent checklist. The personality file does not determine what is in scope; the requirements do.

## Principles vs. Requirements Scope

Some documents are **orientation documents** — they name responsibilities, reserve architectural territory, and introduce domain concepts that span MVP and beyond. Read them for context. Do not derive findings from them:

- `principles.md` files (bedrock, phase-level, and subsystem-level)
- `requirements/conceptual-model.md`
- `design/architecture.md`

A concept named in an orientation document has no design obligation until a corresponding MVP-tagged `R:` row exists. The scope gate above is the only test.

**Actual design files are not exempt.** Design coverage in `design/data-model.md`, `design/engine/`, `design/db/`, `design/controller/`, and `design/ux/` must be anchored to an MVP-tagged requirement. Design that exists without a corresponding `MVP`-tagged `R:` row is unanchored — that is a legitimate finding.
