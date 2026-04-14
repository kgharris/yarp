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
    /// Reads all plan files, runs any pending schema migrations, validates the
    /// resulting plan graph, and returns a fully-resolved `Plan` ready for
    /// engine consumption. The caller receives a clean `Plan` or a typed error
    /// — no raw or unversioned data escapes the trait boundary.
    load(dir: &Path) -> Result<Plan, PlanStoreError>

    /// Persist a plan to the given directory.
    ///
    /// The directory must not already exist. Creates the directory and writes
    /// all plan files. Returns Ok(()) on success. Any filesystem failure
    /// returns a PlanStoreError::Io with the OS error.
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

`PlanViolation` carries the specific validation failure — no members, no
accounts, missing required assumption, allocation sum error — in enough detail
for the caller to produce the error messages defined in
[design/ux/cli.md](../ux/cli.md#error-format).

---

## Implementations

| Type | Use Case |
|---|---|
| `JsonPlanStore` | Production — reads/writes JSON files in the plan directory |
| `MemoryPlanStore` | Testing — no filesystem; plan constructed in memory |

Both implement `PlanStore`. The Model facade binds to the trait; the concrete
type is supplied at construction time and never exposed to callers above the facade.

---

## Write Atomicity

`JsonPlanStore` writes each file to a temporary path in the same directory then
renames atomically. A crash mid-write leaves the prior file intact. No partial
writes are visible to readers.

`MemoryPlanStore` has no atomicity concern — all state is in memory.
