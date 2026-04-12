# Architecture

## Overview

YARP is a desktop retirement planning application that projects household net
worth over a multi-decade timeline. It is a Tauri application: a Rust backend
hosts the computation engine and persistence layer; a React/TypeScript frontend
renders the plan. The system is organized around MVC — the engine computes,
the controller mediates, the view renders.

---

## Technology Stack

| Layer | Technology | Role |
|-------|-----------|------|
| Engine (Model) | Rust | Projection computation, financial logic, stream evaluation |
| DB (Model) | Rust + JSON / SQLite | Plan persistence; backend is swappable behind a trait |
| Controller | Rust (Tauri commands) | Request validation, engine orchestration, IPC serialization |
| UX (View) | React + TypeScript | Rendering; receives view-ready payloads from the controller |
| IPC Bridge | Tauri IPC | Transport between Controller and UX |

---

## System Layers

### Engine

- Pure computation: given plan data and a target denomination, produces projection output. No side effects.
- Evaluates streams year by year across the full plan timeline.
- All internal monetary arithmetic uses arbitrary-precision decimal; no rounding.
- The sole site of denomination conversion. Accepts a target denomination parameter and returns output in that frame.
- Modules do not call each other. The Controller sequences the pipeline.
- No awareness of requests or serialization formats.

### DB

- Reads and writes plan data. No computation.
- Exposes a `PlanStore` trait; the Model facade is the only consumer.
- Production backend: JSON files (human-readable, diffable). Test backend: in-memory.
- Owns schema versioning and migration; no layer above the DB ever sees unversioned data.
- The Controller never holds a direct reference to the DB layer.

### Controller

- Owns the command surface: receives IPC requests, validates inputs, shapes responses.
- Orchestrates the engine pipeline — determines which modules run and in what order.
- Forwards the UX's requested denomination to the Engine as a parameter; does not convert denominations itself.
- Manages scenario state: which scenario is active, overlay resolution.
- Calls the Model facade for persistence; never accesses DB directly.
- No financial computation. No rendering.

### UX

- Renders view-ready payloads delivered by the Controller.
- Zero financial logic. Dollar values are received, not computed.
- Display preferences (denomination toggle, active scenario, navigation state) are
  UI state owned by the View, not financial state.
- All visual properties (color, spacing, typography) are defined once in a central
  style specification and referenced by name — never inlined in component specs.
- Every interactive component spec must cover all states: default, hover, active,
  disabled, loading, error, empty.

---

## Data Flow

A user interaction (e.g., changing a contribution assumption) triggers an IPC
command from the UX to the Controller. The Controller validates the request and
calls the Model facade to persist the updated assumption in YZV. It then
orchestrates the engine pipeline: the engine reads the updated plan data and
re-evaluates the affected streams, producing YNV projection output for each year.
The UX specifies the desired denomination (default: CNV) in the request. The
Controller forwards this to the engine, which converts and returns output in the
requested denomination. The Controller serializes it into a view-ready payload
and returns it over IPC. The UX receives
the payload and re-renders the affected views. No financial computation occurs
outside the engine; no raw model data is read directly by the UX.

---

## Key Invariants

- **Denomination is always explicit.** Every dollar amount belongs to exactly one
  frame: YZV, CNV, YNV, or PNV. An amount without a denomination is a bug.
- **Storage is always YZV.** All persisted configuration and assumption values are
  in year-zero dollars. The anchor year is fixed at plan creation.
- **Engine output is in the requested denomination.** The UX specifies the target
  frame with each request; the Engine performs the conversion and returns output
  in that frame.
- **Display defaults to CNV.** Users see current-year purchasing power unless they
  switch denomination modes.
- **Historicals are PNV.** Observed actuals are recorded in the nominal dollars of
  their reference date and are immutable once committed.
- **Every time-varying quantity is a stream.** Streams span the full timeline and
  encode their own boundary logic. Consumers iterate; they do not check whether a
  stream is active.
- **Business rules are data.** Tax brackets, IRS limits, RMD tables, SS parameters —
  all delivered to the engine as streams. Algorithms operate on the data; they do
  not embed rule values.
- **Engine modules do not call each other.** The Controller sequences the pipeline.
- **The DB is Model-internal.** The Controller calls the Model facade; it never
  holds a DB reference.

---

## Artifact Map

| Concern | Where to look |
|---------|--------------|
| Domain entities and stream model | [requirements/conceptual-model.md](../requirements/conceptual-model.md) |
| Engine requirements | [requirements/engine/](../requirements/engine/) |
| UX requirements | `requirements/ux/` (when written) |
| Data model (stream schema, lifecycle events, account types) | [design/data-model.md](data-model.md) |
| Engine design principles | [design/engine/principles.md](engine/principles.md) |
| DB design principles | [design/db/principles.md](db/principles.md) |
| Controller design principles | [design/controller/principles.md](controller/principles.md) |
| UX design principles | [design/ux/principles.md](ux/principles.md) |
| Cross-cutting design principles | [design/principles.md](principles.md) |
| Bedrock principles (MVC, streams, denomination, data-driven rules) | [bedrock-principles.md](../bedrock-principles.md) |
| Reviewer personalities and workflows | `ai/personalities/`, `ai/workflows/` |
