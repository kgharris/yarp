# DB — Design Principles

The database layer is the **persistence** part of the Model. See
[../../bedrock-principles.md](../../bedrock-principles.md) for the MVC principle
this enforces.

## MVC Constraints

- DB is responsible for reading and writing plan data — nothing more.
- No computation happens here. The DB layer stores and retrieves; the Engine computes.
- No awareness of API request shape or response format.
- Schema changes are driven by data requirements, never by display or routing needs.

## Model-Internal Persistence

The DB layer is internal to the Model. The Controller never interacts with
persistence directly — it calls the Model facade for operations like loading
a plan or saving results. The Model facade owns the DB dependency and delegates
to it internally.

This means:
- The `PlanStore` trait is a Model-internal interface, not a Controller dependency
- The Controller does not hold a `Box<dyn PlanStore>` or equivalent
- The Model facade is the only consumer of the persistence trait
- Swapping persistence backends (JSON, in-memory, SQLite) requires no Controller changes
  because the Controller never knew about the backend in the first place

## Trait-Based Backend Abstraction

The DB layer exposes a trait-based interface so the persistence backend is
replaceable without changes to the Model facade or anything above it.

| Variant | Use Case |
|---|---|
| `JsonPlanStore` | Production — human-readable, diffable, git-friendly |
| `MemoryPlanStore` | Testing — no filesystem, fully isolated |
| `SqlitePlanStore` | Future — if query performance or atomicity becomes a requirement |

## Schema Versioning

The DB layer owns schema migration. On startup, the persistence backend reads
version metadata and runs any pending migrations before returning control.
No layer above the DB ever sees unversioned data.
