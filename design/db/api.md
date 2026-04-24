# DB — PlanStore Trait

The `PlanStore` trait is the Model-internal persistence interface. The Model
facade is its only consumer. No caller above the facade ever holds a direct
`PlanStore` reference.

---

## PlanStore

```
PlanStore {
    /// Load a plan from the given directory.
    ///
    /// Reads all plan files, validates the plan graph, and returns a
    /// fully-resolved `Plan` ready for engine consumption. The caller receives
    /// a clean `Plan` or a typed error — no raw or unversioned data escapes
    /// the trait boundary.
    load(dir: &Path) -> Result<Plan, PlanStoreError>

    /// Persist a plan to the given directory.
    ///
    /// If the directory does not exist, creates it. If it already exists,
    /// overwrites plan files in place using atomic writes. Returns Ok(())
    /// on success. Any filesystem failure returns a PlanStoreError::Io
    /// with the OS error.
    save(dir: &Path, plan: &Plan) -> Result<(), PlanStoreError>
}
```

---

## PlanStoreError

```
PlanStoreError =
    | NotFound(path: Path)          -- directory does not exist
    | InvalidStore(path: Path)      -- directory exists but is not a valid plan store
    | SchemaMismatch {              -- plan schema version is incompatible
        found:    SchemaVersion,
        expected: SchemaVersion,
      }
    | InvalidPlan(violations: Vec<PlanViolation>)  -- plan fails validation
    | Io(source: OsError)           -- filesystem error; message is OS error verbatim
```

`PlanViolation` carries the specific validation failure in domain terms:
violation kind, affected entity ID, and measured value (e.g., no members, no
accounts, missing required assumption, allocation sum deviating from 1.0).

---

## Implementations

| Type | Use Case |
|---|---|
| `JsonPlanStore` | Production — reads/writes JSON files in the plan directory |
| `MemoryPlanStore` | Testing — no filesystem; plan constructed in memory |

Both implement `PlanStore`. The Model facade binds to the trait; the concrete
type is supplied at construction time and never exposed to callers above the facade.

`MemoryPlanStore` exists solely for design-for-testability. Its construction
interface is not specified here — test harnesses will construct and seed it in
whatever way is convenient for the test scenario. Only the `PlanStore` trait
contract matters; the internal structure is an implementation detail of the
test infrastructure.

---

## Write Atomicity

`JsonPlanStore` writes each file to a temporary path in the same directory then
renames atomically. A crash mid-write leaves the prior file intact. No partial
writes are visible to readers.

`MemoryPlanStore` has no atomicity concern — all state is in memory.
