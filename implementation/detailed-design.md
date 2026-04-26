# Project Detailed Design

This document covers project-wide implementation-level decisions: workspace
layout, crate structure, dependencies, conventions, and the CLI binary.
Component-specific detailed designs are in their respective
directories:

- [Engine detailed design](engine/detailed-design.md) — projection engine,
  core types, procedures, evaluation, validation, generate

---

## Crate Structure

A Cargo workspace with two crates: `yarp-core` (library) and `yarp-cli`
(binary). No workspace split between engine and DB.

The single-crate decision is not an MVP shortcut — it is architecturally
correct for the full target system including the future Tauri desktop app.
When the Tauri app ships, the workspace grows by one binary crate:

```
yarp-cli   → yarp-core
yarp-tauri → yarp-core
```

Both binaries consume `yarp-core` through the same public API. The Controller
layer (Tauri commands) lives in `yarp-tauri`, not in a separate library crate.
The View is React/TypeScript, entirely outside Rust. No third Rust library crate
needs the types independently — so a `yarp-types` crate has no consumer that
`yarp-core` doesn't already serve. A `yarp-model` crate would extract 3 methods
and force unnecessary indirection. Neither split earns its keep.

### Workspace Layout

```
yarp/
  Cargo.toml                          # workspace root
  crates/
    yarp-core/                        # library crate -- engine + db
      Cargo.toml
      src/
        lib.rs                        # public API re-exports: Model, ModelError, Projection, etc.
        types/
          mod.rs                      # re-exports all type submodules
          ids.rs                      # YARP_NAMESPACE, UUID v5 derivation functions
          denomination.rs             # PointDenomination, ValueUnit
          stream.rs                   # Stream, StreamTemplate, StreamKind, StreamStart, TerminationRef, ValueSchema, AttributeDef
          procedure.rs                # StreamProcedure enum (type definition only, not dispatch logic)
          point.rs                    # StreamPoint, PointEntry
          entities/
            mod.rs                    # re-exports all entity submodules
            plan.rs                   # Plan, Household
            member.rs                 # Member, MemberRole, LifecycleEvent, EventKind
            account.rs                # Account, AccountOwner, HsaCoverageType, FilingStatus
            property.rs               # Property
            vehicle.rs                # Vehicle
            liability.rs              # AmortizedLoan, InterestOnlyLoan, CreditLine
          owned_by.rs                 # OwnedBy enum
          plan_graph.rs               # PlanGraph aggregate (serializable graph, distinct from Plan entity)
          plan_context.rs             # PlanContext (PlanGraph + derived caches, e.g. CpiFactors)
          errors.rs                   # ModelError, PlanViolation
        store/
          mod.rs                      # PlanStore trait + re-exports
          memory.rs                   # MemoryPlanStore (test backend)
          json.rs                     # JsonPlanStore (production backend)
        engine/
          mod.rs                      # re-exports; no public surface beyond Model
          timeline.rs                 # timeline derivation (called at load time, cached on PlanContext)
          tree.rs                     # Phase 2: projection tree construction
          eval.rs                     # Phase 3: eval(), sweep loop, MemoTable
          procedures/
            mod.rs                    # dispatch() function
            stored.rs                 # Stored procedure
            additive.rs               # Additive procedure
            product.rs                # Product procedure
            end_of_year_growth.rs     # EndOfYearGrowth base case + recurrence
            amortized_schedule.rs     # AmortizedSchedule base case + recurrence
            interest_only.rs          # InterestOnly procedure
          convert.rs                  # yzv_to_ynv
          lifecycle.rs                # MemberLifecycleStream resolution
          resolve.rs                  # resolved_start, resolved_end, identity_value
          stored_point.rs             # stored_point_value, cursor logic
          validate/
            mod.rs                    # top-level validate() that runs all checks
            denomination.rs           # checks 1, 2
            allocation.rs             # check 4
            members.rs                # checks 5, 6, 10
            streams.rs                # checks 7, 8, 11, 13, 14
            carry_forward.rs          # check 12
            labels.rs                 # check 15
            structural.rs             # has_members, has_accounts, cpi_exists, input_refs, event_refs, required_slots, commas, effective_rate_tree
          generate.rs                 # generate() pipeline
        model.rs                      # Model facade (associated functions)
      tests/                            # integration tests (crate-external, use pub API only)
        common/
          mod.rs                      # shared test helpers: fixture loader, plan_builder, result comparator
          plan_builder.rs             # PlanBuilder helper
          fixture.rs                  # TOML fixture parser (independent of PlanStore)
        projection_tests.rs           # data-driven golden-file tests for projection scenarios
        validation_tests.rs           # invalid-plan fixtures exercising each validator
        generate_tests.rs             # generate -> load -> validate -> project round-trip
        cli_tests.rs                  # CLI output format verification (table, CSV, JSON)
      tests/fixtures/                   # data-driven test cases
        canonical-example.toml        # inputs for the data-model numerical example
        canonical-example.expected    # hand-verified expected results
        single-account-fixed-rate.toml
        single-account-fixed-rate.expected
        amortized-loan-payoff.toml
        amortized-loan-payoff.expected
        ...                           # one .toml + .expected pair per scenario
    yarp-cli/                         # binary crate
      Cargo.toml
      src/
        main.rs                       # clap arg parsing, mode dispatch, output formatting
```

**Unit tests** live inline in each source module as `#[cfg(test)] mod tests`.
Every source file in `src/` has a corresponding test module at the bottom.
Unit tests exercise the module's functions directly, using `pub(crate)`
visibility. They use `PlanBuilder` for constructing inputs where a full fixture
would be overkill.

**Integration tests** live in `crates/yarp-core/tests/`. These are
crate-external — they can only access the `pub` API surface of `yarp-core`.
Integration tests are data-driven: each test reads a TOML fixture file using
the independent fixture parser (not `PlanStore::load()`), constructs a
`PlanGraph`, runs the projection, and compares against the `.expected` file.

**Shared test helpers** live in `tests/common/`. The fixture parser, the
`PlanBuilder`, and the result comparison logic are shared across all integration
test files. `tests/common/mod.rs` re-exports them.

**Module granularity rationale.** Source modules are split to keep each file to
a single focused responsibility with a proportionate inline test block. In
particular: `entities/` is split by domain entity (member, account, property,
vehicle, liability) rather than one monolithic file; `procedures/` has one file
per procedure variant; `validate/` has one file per invariant group. Do not
consolidate these into larger files — the inline unit test pattern depends on
each file being small enough that the test block remains readable alongside the
implementation.

### Cargo.toml Files

**Workspace root** (`yarp/Cargo.toml`):

```toml
[workspace]
resolver = "2"
members = ["crates/yarp-core", "crates/yarp-cli"]

[workspace.dependencies]
rust_decimal = "1"
rust_decimal_macros = "1"
uuid = { version = "1", features = ["v5"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
indexmap = { version = "2", features = ["serde"] }
thiserror = "2"
clap = { version = "4", features = ["derive"] }
```

Shared dependencies are declared once at workspace level and inherited by member
crates. This keeps versions consistent and avoids drift.

**Library crate** (`crates/yarp-core/Cargo.toml`):

```toml
[package]
name = "yarp-core"
version = "0.1.0"
edition = "2021"

[dependencies]
rust_decimal = { workspace = true }
rust_decimal_macros = { workspace = true }
uuid = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
indexmap = { workspace = true }
thiserror = { workspace = true }

[dev-dependencies]
proptest = "1"
tempfile = "3"
```

**Binary crate** (`crates/yarp-cli/Cargo.toml`):

```toml
[package]
name = "yarp-cli"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "yarp-cli"
path = "src/main.rs"

[dependencies]
yarp-core = { path = "../yarp-core" }
clap = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
chrono = "0"
```

The binary is named `yarp-cli` to distinguish it from the future Tauri GUI
application.

### Module Dependency Rules

These are acyclic and strictly enforced:

- `types` is a leaf — no dependencies on other yarp-core modules.
- `store` depends on `types` only.
- `engine` depends on `types` only. It never imports from `store`.
- `model` depends on `types`, `store`, and `engine` — it is the composition root.

The `engine` module never touches persistence; the `store` module never touches
computation. `Model` is the sole site where both are composed.

### Visibility Rules

The architectural boundary between engine, store, and the public API surface is
enforced by visibility, not crate boundaries:

- `types/` — `pub` (visible to all dependents)
- `model.rs` — `pub` (the facade)
- `engine/` — `pub(crate)` (invisible outside `yarp-core`)
- `store/` — `pub(crate)` (invisible outside `yarp-core`)

With this visibility, `yarp-cli` (or any future dependent like `yarp-tauri`)
sees only `Model`, `ModelError`, `Projection`, `GeneratePlanParams`, and the
type definitions. It cannot import `eval()`, `JsonPlanStore`, `PlanStore`, or
any engine/store internal. `Model`'s public associated functions (`load`,
`generate`, `get_projection`) use `JsonPlanStore` internally — callers never
name the store type. The module boundary enforces the MVC contract at compile
time — the same guarantee a crate boundary would provide, without the overhead.

The one rule `pub(crate)` does not enforce is that `engine` never imports from
`store`. That is a module dependency rule enforced by convention. If it is
violated, splitting into separate crates is a mechanical refactor — extract
`types/` into `yarp-types`, have `yarp-core` re-export it, dependents don't
change.

**When to revisit:** split into crates if (a) a third Rust library crate needs
the types independently, (b) the engine module starts importing from store, or
(c) compile times become a bottleneck. None of these are likely at MVP or Tauri
phase.

### Rust Idiom Notes

Module directories use `mod.rs` files (e.g., `types/mod.rs`, `engine/mod.rs`).
The alternative convention (`types.rs` alongside a `types/` directory) is also
valid Rust but `mod.rs` is used here for consistency with the directory-based
layout throughout.

`lib.rs` re-exports the public API surface. Internal modules (`engine`, `store`)
are `pub(crate)` — only `model.rs` and `types/` are `pub`. The library's public
surface is intentionally narrow: `Model` (unit struct with associated
functions), `ModelError`, `Projection`, `GeneratePlanParams`, `PlanContext`,
`PlanGraph`, and the type definitions needed to construct and inspect them.
`PlanStore`, `JsonPlanStore`, and `MemoryPlanStore` are not public — `Model`
encapsulates the store internally.

---

## Dependencies

| Crate | Purpose |
|-------|---------|
| `rust_decimal` | Fixed 96-bit significand (28-29 significant digits). Sufficient for 40-year projections. Chosen over `bigdecimal` for performance; chosen over `f64` per [implementation principles](engine/principles.md) (no floating point for monetary values). |
| `rust_decimal_macros` | `dec!()` macro for readable test literals (e.g., `dec!(10000)` instead of `Decimal::from(10000)`). |
| `uuid` | UUID v5 (SHA-1 namespace) generation. The `v5` feature flag enables `Uuid::new_v5()`. |
| `serde` / `serde_json` | Serialization for `JsonPlanStore`. All entity and stream types derive `Serialize`/`Deserialize`. |
| `indexmap` | Insertion-order-preserving maps for `Stream.inputs`, `StreamPoint.entries`, and entity collections on `PlanGraph`. The `serde` feature enables serialization. |
| `thiserror` | Derive `Error` + `Display` for `ModelError` and `PlanStoreError`. Reduces boilerplate. |
| `clap` | CLI argument parsing (`yarp-cli` only). The `derive` feature enables declarative argument structs. |
| `proptest` | (dev-only) Property-based testing for numerical invariants. |
| `cargo-llvm-cov` | (dev tool) Line and branch coverage reporting. |

---

## PlanStore Trait

```rust
pub(crate) trait PlanStore {
    fn load(&self, dir: &Path) -> Result<PlanGraph, PlanStoreError>;
    fn save(&self, dir: &Path, plan: &PlanGraph) -> Result<(), PlanStoreError>;
}
```

`PlanStore` and its implementations are `pub(crate)` — internal to `yarp-core`.
`Model`'s public associated functions (`load`, `generate`) use `JsonPlanStore`
internally. Test entry points (`load_with`, `generate_with`) accept any
`PlanStore` implementation.

Two implementations:

- `JsonPlanStore` — production backend. File layout and schema versioning are
  DB-layer concerns, specified in [design/db/](../design/db/).
- `MemoryPlanStore` — test backend. Holds a `PlanGraph` in memory, `save()`
  clones, `load()` returns the clone.

### PlanStoreError

```rust
#[derive(Debug, thiserror::Error)]
enum PlanStoreError {
    #[error("{0}")]
    NotFound(PathBuf),

    #[error("{0}")]
    InvalidStore(PathBuf),

    #[error("schema version {found}; expected {expected}")]
    SchemaMismatch { found: String, expected: String },

    #[error("{0:?}")]
    InvalidPlan(Vec<PlanViolation>),

    #[error("{0}")]
    Io(#[from] std::io::Error),
}
```

`ModelError` maps from `PlanStoreError` — each variant has a 1:1 mapping.
`Model` converts at the facade boundary; callers above never see
`PlanStoreError`. `SchemaVersion` is a `String`, not a `u32`.

---

## CLI Integration

See [implementation/ux/cli/detailed-design.md](ux/cli/detailed-design.md) for
the authoritative CLI specification — argument parsing, output formats, mode
dispatch, row ordering, error handling, and formatter design.

---

## Testing Strategy

Test-driven development. Tests are written before the code they exercise.
Every function, procedure, and integration point gets a test before its
implementation. Integration tests are written before the glue code that
makes integration possible.

### Dependencies (dev-only)

- `proptest` -- property-based testing for numerical invariants.
- `cargo-llvm-cov` -- line and branch coverage measurement.

### Coverage Requirements

Unit tests must achieve 100% line coverage. Critical branches must also
have branch-decision coverage (both arms exercised). Branch-decision
coverage does not need to be exhaustive across all modules, but must cover
the paths where a wrong branch produces silently incorrect results.

### Data-Driven Golden-File Tests

Test cases are data-driven. Each test case is a TOML config file that specifies
inputs and points to an expected results file. The test runner:

1. Reads the config file using its own parser (not `PlanStore::load()`)
2. Constructs a `PlanGraph` from the specified inputs
3. Runs the projection
4. Compares output against the expected results file

**The test framework must not use `PlanStore::load()` to read test inputs.**
Using `load()` would make tests depend on the code under test. The test config
format has its own parser -- deliberately simple, independent of the plan
persistence layer.

Expected values are hand-verified against manual calculations, not captured
from code output. The canonical numerical example from
[data-model.md](../design/data-model.md) ($10,000 seed, 6.4% rate, $1,000
contribution, 3%/4% CPI -> $10,000, $11,670, $13,488.08) is the primary golden
test.

Test fixture layout:

```
crates/yarp-core/tests/
  fixtures/
    single-account-fixed-rate.toml      -- test config
    single-account-fixed-rate.expected  -- expected results
    canonical-example.toml
    canonical-example.expected
    amortized-loan-payoff.toml
    amortized-loan-payoff.expected
    ...
```

### PlanBuilder

A test helper struct that constructs valid `PlanGraph` instances with sensible
defaults. Methods like `.with_member(name, birth_year)`,
`.with_account(label, kind, owner, opening_balance)`,
`.with_contribution(label, member, account, amount)` chain to build up a plan.
`.build()` returns a valid `PlanGraph`. Used in unit tests where a full
data-driven fixture would be overkill.

### Rejected: Snapshot Tests (insta)

Automatic snapshot capture tools like `insta` are not used. Reasons:

1. Snapshots capture what the code *does*, not what it *should do* -- a bug
   present at capture time is locked in as the expected output.
2. This project is pre-MVP and output formats will churn, causing constant
   snapshot regeneration that becomes a rubber-stamp exercise.
3. The value is redundant with hand-verified golden-file tests, which are
   anchored to known-correct calculations.

### Rounding Convention

Carry full `Decimal` precision through all computation. Round to 2 decimal
places only in `collect_projection_output` when building output
`StreamPoint.entries`. All intermediate values are exact.

---

