# DB Detailed Design

## Overview

This document is the detailed design for the DB (persistence) component of
`yarp-core`. It captures implementation-level decisions -- data structures, type
signatures, serialization strategy, and load/save pipelines -- at a level below
[design/db/](../../design/db/) but above individual source files.

The DB component is the persistence layer within the Model. It reads and writes
`PlanGraph` data to and from storage backends. It performs no computation, no
validation beyond format compatibility checks, and has no awareness of the
engine or any layer above the Model facade. Its sole consumer is `Model` in
`model.rs`.

For workspace layout, crate structure, dependencies, CLI integration, and build
sequence, see the [project detailed design](../detailed-design.md). For the
engine's `PlanGraph`, `PlanContext`, and `Model` facade, see the
[engine detailed design](../engine/detailed-design.md).

Authoritative design specifications:

- [api.md](../../design/db/api.md) -- PlanStore trait, PlanStoreError,
  write atomicity
- [json-plan-store.md](../../design/db/json-plan-store.md) -- JSON format,
  directory layout, schema versioning
- [db principles](../../design/db/principles.md) -- MVC constraints,
  trait-based abstraction
- [data-model.md](../../design/data-model.md) -- entity schemas, stream
  primitives, design invariants

The store lives in the `store/` module of `yarp-core`:

```
store/
  mod.rs      # PlanStore trait, PlanStoreError, SCHEMA_VERSION constant, re-exports
  json.rs     # JsonPlanStore struct + PlanStore impl
  memory.rs   # MemoryPlanStore struct + PlanStore impl
```

---

## Module Dependency and Visibility

The module dependency rules are defined in the
[project detailed design](../detailed-design.md#module-dependency-rules). The
store-specific rule is:

**`store` depends on `types` only.** It never imports from `engine`. CPI
computation, validation, denomination conversion -- all engine concerns -- are
unreachable from `store/`. `Model` in `model.rs` is the composition root where
store and engine are brought together.

All items in `store/` are `pub(crate)`. The `store` module is invisible outside
`yarp-core`. `mod.rs` re-exports `PlanStore`, `PlanStoreError`, `JsonPlanStore`,
and `MemoryPlanStore` so that `model.rs` imports from `crate::store::*`.

`PlanStoreError` lives in `store/mod.rs`, not in `types/errors.rs`. It is a
store-layer error, not a shared type. `ModelError` (which wraps it) lives in
`types/errors.rs` -- that separation is deliberate. The store produces
`PlanStoreError`; `Model` converts it to `ModelError` at the facade boundary.

---

## PlanStore Trait and PlanStoreError

### PlanStore

```rust
pub(crate) trait PlanStore {
    fn load(&self, dir: &Path) -> Result<PlanGraph, PlanStoreError>;
    fn save(&self, dir: &Path, plan: &PlanGraph) -> Result<(), PlanStoreError>;
}
```

`Model` calls `PlanStore` methods internally via its `load_with` and
`generate_with` associated functions. The default `load()` and `generate()`
use `JsonPlanStore`; tests use `load_with(&MemoryPlanStore::new(), dir)`. The
trait returns `PlanGraph` (the serializable aggregate), not `Plan` (the entity
record). The design spec uses `Plan` but the project DD refines this to
`PlanGraph`; the project DD governs at the implementation level.

For the `Model` facade definition and its methods, see the
[engine detailed design](../engine/detailed-design.md#model-facade-api).

### PlanStoreError

```rust
#[derive(Debug, thiserror::Error)]
pub(crate) enum PlanStoreError {
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

`From<PlanStoreError> for ModelError` maps each variant 1:1:

```rust
impl From<PlanStoreError> for ModelError {
    fn from(e: PlanStoreError) -> Self {
        match e {
            PlanStoreError::NotFound(p)    => ModelError::NotFound(p),
            PlanStoreError::InvalidStore(p) => ModelError::InvalidStore(p),
            PlanStoreError::SchemaMismatch { found, expected } =>
                ModelError::SchemaMismatch { found, expected },
            PlanStoreError::InvalidPlan(v) => ModelError::InvalidPlan(v),
            PlanStoreError::Io(e)          => ModelError::Io(e),
        }
    }
}
```

The `InvalidPlan` variant exists on the trait for the contract but is never
constructed by either store implementation in MVP. Validation runs in
`Model::load()`, which constructs `ModelError::InvalidPlan` directly, not via
the store.

### Error Variant Trigger Conditions

| Scenario | Error variant |
|----------|--------------|
| Directory does not exist | `NotFound(dir)` |
| Path exists but is not a directory | `InvalidStore(dir)` |
| Directory exists but `plan.json` is missing | `InvalidStore(dir)` |
| `plan.json` contains invalid JSON | `Io(std::io::Error::new(InvalidData, serde_err))` |
| `plan.json` has wrong schema version | `SchemaMismatch { found, expected }` |
| Filesystem read/write/rename failure | `Io(source)` |
| Parent directory missing on save | `Io(source)` (from `fs::create_dir`) |
| Save path exists but is not a directory | `Io(std::io::Error::new(InvalidInput, "path exists but is not a directory"))` |

JSON parse errors (serde failures) map to `Io` using
`std::io::Error::new(ErrorKind::InvalidData, serde_err)`. A corrupt JSON file
is a data integrity failure, not "an invalid store" (which means the directory
structure is wrong, e.g., missing `plan.json`). The `Io` variant with
`InvalidData` produces the serde error message, which tells the user what is
wrong with the file content.

---

## Schema Versioning

### Constant

```rust
// store/mod.rs
pub(crate) const SCHEMA_VERSION: u32 = 1;
```

The schema version is a `u32`, consistent with the design spec
([json-plan-store.md](../../design/db/json-plan-store.md)) which shows
`"schema_version": 1` as an integer. In JSON, the value serializes as the
integer `1`.

### PlanFile Struct

Two variants of `PlanFile` are used -- one for deserialization (load) and one
for serialization (save).

**Load (owns the PlanGraph):**

```rust
#[derive(Deserialize)]
struct PlanFile {
    #[serde(deserialize_with = "flexible_version")]
    schema_version: u32,
    plan: PlanGraph,
}
```

**Save (borrows the PlanGraph):**

```rust
#[derive(Serialize)]
struct PlanFile<'a> {
    schema_version: u32,
    plan: &'a PlanGraph,
}
```

The save variant borrows `&PlanGraph` rather than cloning, avoiding a full
graph copy on every save. Both structs are private to `store/json.rs`.

**Flexible deserialization:** `flexible_version` is a custom serde
deserializer that accepts both JSON integer `1` and JSON string `"1"`,
normalizing to `u32`. This ensures hand-edited files work regardless of
whether the user writes the version as a number or a quoted string.

The version check uses direct comparison against the constant:

```rust
if plan_file.schema_version != SCHEMA_VERSION {
    return Err(PlanStoreError::SchemaMismatch {
        found: plan_file.schema_version.to_string(),
        expected: SCHEMA_VERSION.to_string(),
    });
}
```

---

## Serde Strategy

### Enum Representation

All enums use serde's default **externally tagged** representation. No
`#[serde(tag = ...)]` attributes on any enum. This is the simplest approach and
correctly handles the mix of unit, newtype, tuple, and struct variants present
in the type definitions.

Internally tagged (`#[serde(tag = "type")]`) was rejected because it does not
support newtype or tuple variants. It would require changing
`OwnedBy::Plan(Uuid)` to `OwnedBy::Plan { plan_id: Uuid }` and
`StreamStart::CalendarYear(i32)` to `StreamStart::CalendarYear { year: i32 }`.
The serde representation must not drive type definition changes.

Adjacently tagged (`#[serde(tag = "type", content = "data")]`) was rejected
because it adds verbosity without benefit over external tagging.

**Serialization examples by variant shape:**

| Shape | Rust | JSON |
|-------|------|------|
| Unit | `MemberRole::Primary` | `"Primary"` |
| Newtype | `OwnedBy::Plan(uuid)` | `{ "Plan": "550e8400-..." }` |
| Tuple | `OwnedBy::MemberAccount(uuid1, uuid2)` | `{ "MemberAccount": ["550e8400-...", "6ba7b810-..."] }` |
| Struct | `StreamProcedure::EndOfYearGrowth { cpi_stream_id }` | `{ "EndOfYearGrowth": { "cpi_stream_id": "550e8400-..." } }` |

### Enum Inventory

All enums use externally tagged representation (serde default -- no attribute
needed):

- `StreamKind` -- unit variants only
- `StreamStart` -- newtype (`CalendarYear`) and struct (`MemberAge`)
- `TerminationRef` -- struct variant (`OnEvent`)
- `StreamProcedure` -- unit (`Stored`, `Additive`, `Product`, `InterestOnly`)
  and struct (`EndOfYearGrowth`, `AmortizedSchedule`)
- `ValueUnit` -- unit variants only
- `PointDenomination` -- unit (`Yzv`) and struct (`Pnv`, `Ynv`)
- `OwnedBy` -- newtype and tuple variants
- `MemberRole` -- unit variants only
- `EventKind` -- unit variants only
- `AccountOwner` -- newtype variants
- `HsaCoverageType` -- unit variants only
- `FilingStatus` -- unit variants only

### Struct-Level Attributes

All entity structs, `Stream`, `StreamPoint`, `PointEntry`, `ValueSchema`,
`AttributeDef`, and `PlanGraph` derive `Serialize` and `Deserialize`:

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
struct Plan {
    id: Uuid,
    name: String,
    anchor_year: i32,
    household_id: Uuid,
}
```

Field-level attributes for `Option<T>` fields:

```rust
#[serde(skip_serializing_if = "Option::is_none")]
#[serde(default)]
closing_year: Option<i32>,
```

Every `Option<T>` field uses both `skip_serializing_if = "Option::is_none"`
(omit from JSON when `None`) and `default` (deserialize as `None` when absent).
This produces clean JSON (absent keys instead of `null`) and round-trips
correctly.

`#[serde(rename_all = "snake_case")]` on structs is a no-op when field names
are already snake_case but serves as documentation-of-intent.

No `#[serde(skip)]` annotations are needed anywhere. `PlanGraph` contains only
serializable fields. `PlanContext` (which holds caches) is never serialized.

### Decimal Handling

`Decimal` values serialize as JSON strings, not numbers. The `serde-str` feature
on `rust_decimal` makes this automatic with no per-field attributes:

```toml
# workspace Cargo.toml [workspace.dependencies]
rust_decimal = { version = "1", features = ["serde-str"] }
```

JSON numbers are IEEE 754 floats and cannot represent exact decimal values. A
round-trip through `f64` would lose precision, undermining the reason
`rust_decimal` was chosen. With `serde-str`, `Decimal` serializes as
`"10000.00"` (string) rather than `10000.00` (number).

### UUID Serialization

The `serde` feature on `uuid` enables `Serialize`/`Deserialize` for `Uuid`.
Without it, compilation fails. UUIDs serialize as hyphenated lowercase strings:

```toml
# workspace Cargo.toml [workspace.dependencies]
uuid = { version = "1", features = ["v5", "serde"] }
```

Example: `"550e8400-e29b-41d4-a716-446655440000"`. This is `uuid`'s default
serde format and requires no per-field attributes.

### Cargo.toml Changes

Two workspace dependency changes are required:

```toml
# workspace Cargo.toml [workspace.dependencies]
rust_decimal = { version = "1", features = ["serde-str"] }   # add serde-str feature
uuid = { version = "1", features = ["v5", "serde"] }         # add serde feature
```

These are workspace-level changes affecting all crates. The current workspace
`Cargo.toml` has `rust_decimal = "1"` (no `serde-str`) and
`uuid = { version = "1", features = ["v5"] }` (no `serde`).

Dev-dependency additions to `yarp-core`:

```toml
[dev-dependencies]
proptest = "1"
tempfile = "3"
```

---

## JSON Document Structure

### Shape of plan.json

`plan.json` is a single JSON object with two top-level fields:

```json
{
  "schema_version": 1,
  "plan": { ... }
}
```

The outer `plan` field is `PlanFile.plan` (the `PlanGraph`). The inner fields
are `PlanGraph`'s fields -- `plan` (the `Plan` entity), `household`, `members`,
etc. The double nesting of `plan` is a natural consequence of the type structure
and does not need to be renamed.

`IndexMap<Uuid, T>` collections serialize as JSON objects keyed by UUID strings.
`points` is `IndexMap<Uuid, Vec<StreamPoint>>` keyed by `stream_id`. The
`stream_id` field on each `StreamPoint` is redundant with the map key but is
kept for self-describing records and consistency checking.

`cpi_factors` does not appear in the JSON. `PlanGraph` has no cache fields;
caches live on `PlanContext`.

### Edge Cases

- Empty collections serialize as empty objects `{}`, not as `null` or absent
  keys.
- Optional fields that are `None` are omitted from JSON (via
  `skip_serializing_if`), not written as `null`.
- `Decimal` values appear as strings: `"10000.00"`.
- UUIDs appear as hyphenated lowercase strings.
- Enum variants use externally tagged representation as shown in the serde
  strategy section.

### Complete Minimal Example

This example shows a minimal valid plan with one member, one 401k account, a CPI
rate stream, one return rate stream, one allocation weight stream, a weighted
return stream, an effective rate root, a balance stream template, one
contribution stream, and the required aggregate templates. Optional fields that
are `None` are omitted.

```json
{
  "schema_version": 1,
  "plan": {
    "plan": {
      "id": "a1b2c3d4-0000-5000-8000-000000000001",
      "name": "My Plan",
      "anchor_year": 2025,
      "household_id": "a1b2c3d4-0000-5000-8000-000000000002"
    },
    "household": {
      "id": "a1b2c3d4-0000-5000-8000-000000000002",
      "plan_id": "a1b2c3d4-0000-5000-8000-000000000001"
    },
    "members": {
      "a1b2c3d4-0000-5000-8000-000000000010": {
        "id": "a1b2c3d4-0000-5000-8000-000000000010",
        "household_id": "a1b2c3d4-0000-5000-8000-000000000002",
        "given_name": "Alice",
        "family_name": "Smith",
        "birth_year": 1995,
        "death_age": 95,
        "role": "Primary"
      }
    },
    "lifecycle_events": {
      "a1b2c3d4-0000-5000-8000-000000000011": {
        "id": "a1b2c3d4-0000-5000-8000-000000000011",
        "plan_id": "a1b2c3d4-0000-5000-8000-000000000001",
        "member_id": "a1b2c3d4-0000-5000-8000-000000000010",
        "kind": "Retirement",
        "age": 65
      }
    },
    "accounts": {
      "a1b2c3d4-0000-5000-8000-000000000020": {
        "id": "a1b2c3d4-0000-5000-8000-000000000020",
        "household_id": "a1b2c3d4-0000-5000-8000-000000000002",
        "owner": {
          "MemberOwned": "a1b2c3d4-0000-5000-8000-000000000010"
        },
        "label": "Alice 401k"
      }
    },
    "properties": {},
    "vehicles": {},
    "amortized_loans": {},
    "interest_only_loans": {},
    "credit_lines": {},
    "streams": {
      "a1b2c3d4-0000-5000-8000-000000000030": {
        "id": "a1b2c3d4-0000-5000-8000-000000000030",
        "kind": "MemberLifecycleStream",
        "owner": {
          "Member": "a1b2c3d4-0000-5000-8000-000000000010"
        },
        "start": {
          "MemberAge": {
            "member_id": "a1b2c3d4-0000-5000-8000-000000000010",
            "age": 0
          }
        },
        "inputs": {},
        "procedure": "Stored",
        "value_schema": {
          "attributes": [
            { "key": "age", "unit": "Age" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000031": {
        "id": "a1b2c3d4-0000-5000-8000-000000000031",
        "kind": "RateStream",
        "label": "cpi",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "start": {
          "CalendarYear": 2025
        },
        "inputs": {},
        "procedure": "Stored",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Rate", "initial": "0.03" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000032": {
        "id": "a1b2c3d4-0000-5000-8000-000000000032",
        "kind": "RateStream",
        "label": "return.large-cap",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "start": {
          "CalendarYear": 2025
        },
        "inputs": {},
        "procedure": "Stored",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Rate", "initial": "0.08" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000033": {
        "id": "a1b2c3d4-0000-5000-8000-000000000033",
        "kind": "RateStream",
        "label": "weight.large-cap",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "start": {
          "CalendarYear": 2025
        },
        "inputs": {},
        "procedure": "Stored",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Rate", "initial": "1.0" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000034": {
        "id": "a1b2c3d4-0000-5000-8000-000000000034",
        "kind": "RateStream",
        "label": "weighted-return.large-cap",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "start": {
          "CalendarYear": 2025
        },
        "inputs": {
          "weight": "a1b2c3d4-0000-5000-8000-000000000033",
          "rate": "a1b2c3d4-0000-5000-8000-000000000032"
        },
        "procedure": "Product",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Rate" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000035": {
        "id": "a1b2c3d4-0000-5000-8000-000000000035",
        "kind": "RateStream",
        "label": "effective-rate",
        "owner": {
          "Account": "a1b2c3d4-0000-5000-8000-000000000020"
        },
        "start": {
          "CalendarYear": 2025
        },
        "inputs": {
          "large-cap": "a1b2c3d4-0000-5000-8000-000000000034"
        },
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Rate" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000036": {
        "id": "a1b2c3d4-0000-5000-8000-000000000036",
        "kind": "DollarStream",
        "label": "balance",
        "owner": {
          "Account": "a1b2c3d4-0000-5000-8000-000000000020"
        },
        "start": {
          "CalendarYear": 2025
        },
        "inputs": {
          "rate": "a1b2c3d4-0000-5000-8000-000000000035",
          "employee-401k": "a1b2c3d4-0000-5000-8000-000000000040"
        },
        "procedure": {
          "EndOfYearGrowth": {
            "cpi_stream_id": "a1b2c3d4-0000-5000-8000-000000000031"
          }
        },
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000040": {
        "id": "a1b2c3d4-0000-5000-8000-000000000040",
        "kind": "DollarStream",
        "label": "employee-401k",
        "owner": {
          "MemberAccount": [
            "a1b2c3d4-0000-5000-8000-000000000010",
            "a1b2c3d4-0000-5000-8000-000000000020"
          ]
        },
        "start": {
          "CalendarYear": 2025
        },
        "terminates": {
          "OnEvent": {
            "event_id": "a1b2c3d4-0000-5000-8000-000000000011"
          }
        },
        "inputs": {},
        "procedure": "Stored",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000050": {
        "id": "a1b2c3d4-0000-5000-8000-000000000050",
        "kind": "DollarStream",
        "label": "net-worth",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "inputs": {
          "assets": "a1b2c3d4-0000-5000-8000-000000000051",
          "liabilities": "a1b2c3d4-0000-5000-8000-000000000054"
        },
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000051": {
        "id": "a1b2c3d4-0000-5000-8000-000000000051",
        "kind": "DollarStream",
        "label": "assets",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "inputs": {
          "retirement": "a1b2c3d4-0000-5000-8000-000000000052",
          "health": "a1b2c3d4-0000-5000-8000-000000000058",
          "taxable": "a1b2c3d4-0000-5000-8000-000000000059",
          "hard-assets": "a1b2c3d4-0000-5000-8000-000000000060"
        },
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000052": {
        "id": "a1b2c3d4-0000-5000-8000-000000000052",
        "kind": "DollarStream",
        "label": "retirement",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "inputs": {
          "Alice 401k": "a1b2c3d4-0000-5000-8000-000000000036"
        },
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000054": {
        "id": "a1b2c3d4-0000-5000-8000-000000000054",
        "kind": "DollarStream",
        "label": "liabilities",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "inputs": {
          "lt-liabilities": "a1b2c3d4-0000-5000-8000-000000000055",
          "st-liabilities": "a1b2c3d4-0000-5000-8000-000000000056"
        },
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000055": {
        "id": "a1b2c3d4-0000-5000-8000-000000000055",
        "kind": "DollarStream",
        "label": "lt-liabilities",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "inputs": {},
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000056": {
        "id": "a1b2c3d4-0000-5000-8000-000000000056",
        "kind": "DollarStream",
        "label": "st-liabilities",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "inputs": {},
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000058": {
        "id": "a1b2c3d4-0000-5000-8000-000000000058",
        "kind": "DollarStream",
        "label": "health",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "inputs": {},
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000059": {
        "id": "a1b2c3d4-0000-5000-8000-000000000059",
        "kind": "DollarStream",
        "label": "taxable",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "inputs": {},
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      },
      "a1b2c3d4-0000-5000-8000-000000000060": {
        "id": "a1b2c3d4-0000-5000-8000-000000000060",
        "kind": "DollarStream",
        "label": "hard-assets",
        "owner": {
          "Plan": "a1b2c3d4-0000-5000-8000-000000000001"
        },
        "inputs": {},
        "procedure": "Additive",
        "value_schema": {
          "attributes": [
            { "key": "value", "unit": "Decimal" }
          ]
        }
      }
    },
    "points": {
      "a1b2c3d4-0000-5000-8000-000000000031": [
        {
          "stream_id": "a1b2c3d4-0000-5000-8000-000000000031",
          "year": 2025,
          "entries": {
            "value": {
              "amount": "0.03",
              "denomination": "Yzv"
            }
          }
        }
      ],
      "a1b2c3d4-0000-5000-8000-000000000032": [
        {
          "stream_id": "a1b2c3d4-0000-5000-8000-000000000032",
          "year": 2025,
          "entries": {
            "value": {
              "amount": "0.08",
              "denomination": "Yzv"
            }
          }
        }
      ],
      "a1b2c3d4-0000-5000-8000-000000000033": [
        {
          "stream_id": "a1b2c3d4-0000-5000-8000-000000000033",
          "year": 2025,
          "entries": {
            "value": {
              "amount": "1.0",
              "denomination": "Yzv"
            }
          }
        }
      ],
      "a1b2c3d4-0000-5000-8000-000000000036": [
        {
          "stream_id": "a1b2c3d4-0000-5000-8000-000000000036",
          "year": 2025,
          "entries": {
            "value": {
              "amount": "50000",
              "denomination": "Yzv"
            }
          }
        }
      ],
      "a1b2c3d4-0000-5000-8000-000000000040": [
        {
          "stream_id": "a1b2c3d4-0000-5000-8000-000000000040",
          "year": 2025,
          "entries": {
            "value": {
              "amount": "5000",
              "denomination": "Yzv"
            }
          }
        }
      ]
    }
  }
}
```

---

## JsonPlanStore

### Struct

```rust
pub(crate) struct JsonPlanStore;
```

`JsonPlanStore` is a zero-sized unit struct. It carries no state. All
configuration (file names, schema version) comes from constants.

### Constants

```rust
// store/json.rs (private)
const PLAN_FILE: &str = "plan.json";
const PLAN_TEMP: &str = "plan.json.tmp";
```

### Load Pipeline

1. Check `dir.is_dir()`. If false: if `dir` does not exist at all
   (`!dir.exists()`), return `NotFound(dir.to_path_buf())`. If it exists but is
   not a directory, return `InvalidStore(dir.to_path_buf())`.
2. Check `dir.join(PLAN_FILE).is_file()`. If false, return
   `InvalidStore(dir.to_path_buf())`.
3. Read file contents via `fs::read_to_string(dir.join(PLAN_FILE))`. On error,
   return `Io(source)`.
4. Deserialize into `PlanFile` via `serde_json::from_str(&contents)`. On serde
   error, return `Io(std::io::Error::new(ErrorKind::InvalidData, e))`.
5. Check `plan_file.schema_version` against `SCHEMA_VERSION`. If mismatch,
   return `SchemaMismatch { found: plan_file.schema_version, expected: SCHEMA_VERSION.to_string() }`.
6. Sort each stream's points by year:
   `plan_file.plan.points.values_mut().for_each(|v| v.sort_by_key(|p| p.year))`.
   This is defensive against hand-edited JSON where points may be out of order.
7. Return `plan_file.plan` (the `PlanGraph`).

**One-phase deserialization:** The load pipeline uses one-phase deserialization
(directly into `PlanFile` struct) for MVP. The design spec says "reads this
field before deserializing `plan`," which favors two-phase. However, with only
one schema version, a two-phase parse adds complexity for no gain. If the JSON
body structure changes in a future version, the serde error will be less clean
than a `SchemaMismatch`, but this is an acceptable MVP trade-off.

**Serde as implicit validation:** Serde deserialization enforces structural
validity before any explicit validation runs. Enum variants must be recognized
(`"Traditional401k"` not `"Foo"`), required fields must be present, and types
must match (a string where an `i32` is expected fails). This provides a
distinct validation layer that catches corrupt or hand-edited files. Explicit
domain validation (the 15 design invariants) runs later in `Model::load()`.

### Save Pipeline

1. If `dir` does not exist, call `fs::create_dir(dir)`. On error, return
   `Io(source)`. Use `create_dir` (NOT `create_dir_all`). `create_dir_all`
   silently creates intermediate directories, masking path errors. The parent
   directory's existence is the caller's responsibility.
2. If `dir` exists but is not a directory, return
   `Io(std::io::Error::new(ErrorKind::InvalidInput, "path exists but is not a directory"))`.
3. Assert each stream's points are sorted by year (`debug_assert`). Sorted
   order is a `PlanGraph` invariant maintained at the point of mutation —
   `load()` sorts on ingest, `generate()` builds in order, and future
   mutation APIs insert in sorted position. `save()` verifies the invariant
   but does not enforce it.
4. Build a `PlanFile<'a>` wrapper:
   ```rust
   let plan_file = PlanFile {
       schema_version: SCHEMA_VERSION,
       plan,
   };
   ```
5. Serialize to pretty JSON via `serde_json::to_string_pretty(&plan_file)`.
   This produces 2-space indentation (serde_json's default).
6. Append a trailing newline (`\n`) to the serialized string:
   ```rust
   let mut json = serde_json::to_string_pretty(&plan_file)
       .map_err(|e| std::io::Error::new(ErrorKind::InvalidData, e))?;
   json.push('\n');
   ```
7. Write to `dir.join(PLAN_TEMP)`. On error, return `Io(source)`.
8. Rename `dir.join(PLAN_TEMP)` to `dir.join(PLAN_FILE)` via `fs::rename`. On
   error, return `Io(source)`.

The atomic temp-file-then-rename pattern handles both new and existing
directories identically. `plan.json.tmp` is written and renamed over any
existing `plan.json`. A crash mid-write leaves the prior `plan.json` intact.

The trailing newline ensures git diffs are clean (no "no newline at end of
file" warning).

**JSON formatting:** Key ordering follows serde's serialization order, which for
`#[derive]` structs is field declaration order. For `IndexMap`, it is insertion
order. Combined with point sorting, this produces deterministic output for the
same input.

---

## MemoryPlanStore

### Struct

```rust
pub(crate) struct MemoryPlanStore {
    plans: RefCell<HashMap<PathBuf, PlanGraph>>,
}
```

Interior mutability via `RefCell` because the `PlanStore` trait takes `&self`.
Keyed by `PathBuf` so different paths store different plans -- mirrors
`JsonPlanStore` semantics.

### Methods

```rust
impl MemoryPlanStore {
    pub(crate) fn new() -> Self {
        MemoryPlanStore {
            plans: RefCell::new(HashMap::new()),
        }
    }

    /// Insert a plan for test setup. Does not go through save() validation
    /// (no directory-already-exists check). This is test infrastructure,
    /// not part of the PlanStore trait.
    pub(crate) fn seed(&self, dir: &Path, plan: PlanGraph) {
        self.plans.borrow_mut().insert(dir.to_path_buf(), plan);
    }
}

impl PlanStore for MemoryPlanStore {
    fn load(&self, dir: &Path) -> Result<PlanGraph, PlanStoreError> {
        self.plans
            .borrow()
            .get(dir)
            .cloned()
            .ok_or_else(|| PlanStoreError::NotFound(dir.to_path_buf()))
    }

    fn save(&self, dir: &Path, plan: &PlanGraph) -> Result<(), PlanStoreError> {
        self.plans.borrow_mut().insert(dir.to_path_buf(), plan.clone());
        Ok(())
    }
}
```

### Clone Semantics

Both `load()` and `save()` clone the `PlanGraph` to ensure full isolation.
Mutations to a loaded `PlanGraph` do not affect the stored copy, and mutations
to the original after saving do not affect the stored copy. This mirrors
`JsonPlanStore`'s behavior where load and save go through serialization
boundaries.

Both `seed()` and `save()` insert-or-overwrite, matching `JsonPlanStore`'s
overwrite semantics.

`MemoryPlanStore` has no concept of `InvalidStore` -- there is no directory
structure to be invalid. The only error path is `NotFound` (key missing on
load).

---

## CPI Factors and Validation Placement

### Where CPI Factors Are Computed

CPI factors are computed in `Model::load()`, never in the store. The store
returns a `PlanGraph` (serializable graph only -- no derived caches).
`Model::load()` computes `CpiFactors`, wraps the graph in a `PlanContext`, and
runs validation.

`PlanGraph` contains no cache fields. `PlanContext` wraps `PlanGraph` with
caches:

```rust
struct PlanContext {
    graph: PlanGraph,
    timeline_start: i32,      // min(member.birth_year) — precomputed at load time
    timeline_end: i32,        // max(member.birth_year + member.death_age) — precomputed at load time
    cpi_factors: CpiFactors,  // precomputed at load time
}
```

The structural separation ensures `PlanGraph` is purely serializable data and
`PlanContext` is graph + derived caches. No `#[serde(skip)]` annotations are
needed because `PlanContext` is never serialized.

### Model::load() Flow

```
Model::load(dir):
    Self::load_with(&JsonPlanStore, dir)

Model::load_with(store, dir):
    let mut graph = store.load(dir)?;                      // returns PlanGraph
    let (timeline_start, timeline_end) =
        engine::timeline::derive_timeline(&graph);         // engine function
    let cpi_stream_id = engine::ensure_cpi_stream(
        &mut graph,
    );                                                     // engine function
    let cpi_factors = engine::convert::compute_cpi_factors(
        cpi_stream_id,
        &graph.points,
        graph.plan.anchor_year,
        timeline_start,
        timeline_end,
    );                                                     // engine function
    let plan_ctx = PlanContext { graph, timeline_start, timeline_end, cpi_factors };
    engine::validate::validate(&plan_ctx)?;                // engine function
    Ok(plan_ctx)
```

The `compute_cpi_factors` function is defined in the engine module. The store
does not import it. `Model` is the composition root where store and engine are
brought together. For the full `compute_cpi_factors` specification, see the
[engine detailed design](../engine/detailed-design.md#cpi-cumulative-product).

### Validation Placement

Validation runs in `Model::load()`, not in the store. The store's only "format
check" is the schema version comparison, which is a format compatibility check,
not domain validation.

The design spec ([api.md](../../design/db/api.md)) says load "validates the plan
graph." This refers to `Model::load()` (the facade method), not
`PlanStore::load()` (the trait method). The design spec was written before the
project DD codified the module dependency rule. The trait-level phrasing is
superseded by the structural constraint: the store cannot import `validate()`
from engine. Validation is domain logic, not persistence logic.

Explicit validation (the 15 design invariants from
[data-model.md](../../design/data-model.md#design-invariants)) runs in
`Model::load()` after CPI computation and `PlanContext` construction, using
`engine::validate::validate()`. Validation failures produce
`ModelError::InvalidPlan(violations)` -- not `PlanStoreError::InvalidPlan`.
The `PlanStoreError::InvalidPlan` variant is structurally present on the trait
but is unused by either store backend in MVP.

---

## DB Testing

The project-wide testing strategy (TDD, coverage requirements, golden-file
tests, PlanBuilder, rejected alternatives, rounding convention) is defined in
the [project detailed design](../detailed-design.md#testing-strategy).

DB tests follow TDD: tests are written before the code they exercise.
Scenarios, fixture data, and expected values are defined authoritatively in
the test artifacts (inline test modules and fixture files), not in this
document.

Tests use `PlanBuilder` for constructing `PlanGraph` instances.
`PlanBuilder::build()` returns a `PlanGraph` directly -- no caches involved.
Use the `tempfile` crate (dev-dependency) for filesystem test isolation.
