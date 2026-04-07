# Testing Context

Artifacts live under [tests/](../../../tests/).

## Structure

- `tests/` mirrors the subsystem structure of the source tree
- Each subsystem has its own test subdirectory

## What Testing Is

Testing verifies that the implementation satisfies requirements and conforms to design.

- Acceptance tests derived from requirements (externally observable behavior)
- Contract tests derived from design (interfaces, data model invariants)
- Unit tests for implementation correctness (functions, modules)
- Integration tests for cross-component behavior (controller ↔ model, model ↔ db)
- Coverage assessment against requirements and design, not just code paths

## What Testing Excludes

- **No new requirements** — tests verify existing requirements, they don't define new behavior
- **No design decisions** — if a test requires a design choice that isn't documented, that's a gap in design, not something to decide in a test
- **No implementation beyond test code** — tests don't fix production code, they surface what's wrong

Test: does every test trace back to a requirement or design contract? If a test exists for something not specified, either the spec has a gap or the test is speculative.

## Altitude

- **Forest:** survey test coverage across all subsystems at breadth; assess whether integration boundaries are tested, not just internals
- **Branch:** read your domain's tests deeply; check that domain-level contracts are asserted, not just unit behavior
- **Leaf:** read your subsystem's tests exhaustively; every behavior specified in requirements and design should have corresponding coverage
