# Implementation Context

Artifacts live under [implementation/](../../../implementation/) and source code
under [src/](../../../src/).

## Structure

- `dependencies.md` — cross-cutting library choices; read by all personalities at this phase
- `engine/` — engine implementation docs and source
- `ux/` — frontend implementation docs and source

## What Implementation Is

Implementation is the concrete working code and the specific technology choices that realize a design.

- Language and framework choices (Rust, TypeScript, React, Tauri)
- Library and dependency selections
- Exact function signatures, type definitions, and class structures
- Coding conventions and style (naming, formatting, module layout)
- Error handling strategies and edge case treatment
- Build configuration, tooling, and deployment

Examples:
- "Use `serde` for JSON serialization with `#[derive(Serialize, Deserialize)]`"
- "React components use shadcn/ui with Tailwind CSS utility classes"
- "JSON files formatted with 2-space indent, sorted keys"
- "Tauri commands return `Result<T, String>` for frontend consumption"

## What Implementation Excludes

- **No business rules or user needs** — "the system must warn when limits are exceeded" is a requirement
- **No data model definitions** — the shape of `TimelineYear` is design
- **No algorithm choices** — which formula to use is design
- **No interface contracts** — what a component exposes and guarantees is design

Test: could this change without affecting the design documents? If not, it's a design decision being made at the wrong layer.

## Altitude

- **Forest:** read `dependencies.md`, then survey each subsystem at breadth for architectural conformance
- **Branch:** read `dependencies.md` and your domain's source deeply
- **Leaf:** read your subsystem's implementation docs and source exhaustively
