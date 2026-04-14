# Engine — Model Facade API

The `Model` trait is the sole entry point into the Model layer for all callers
— the CLI, the Controller, and test harnesses. No caller ever holds a reference
to `JsonPlanStore`, `MemoryPlanStore`, or any other backend directly.

See [design/db/api.md](../db/api.md) for the `PlanStore` trait that the Model
facade delegates to internally.

---

## Model

```
Model {
    /// Load a plan from a directory. Delegates to PlanStore::load(), which
    /// runs schema migration and validation before returning. Callers receive
    /// a clean Plan or a typed error — no raw or unversioned data escapes
    /// this boundary.
    load(dir: &Path) -> Result<Plan, ModelError>

    /// Generate a minimal runnable plan and write it to a new directory.
    ///
    /// Constructs the Plan graph from GeneratePlanParams, then delegates to
    /// PlanStore::save() to persist it. The directory must not already exist.
    generate(dir: &Path, params: GeneratePlanParams) -> Result<(), ModelError>
}
```

`Plan` is the deserialized, fully-validated in-memory plan graph — the data
structure the engine consumes. `Model::generate()` is responsible for
constructing the minimal `Plan`; `PlanStore::save()` is responsible for
writing it to disk.

---

## GeneratePlanParams

```
GeneratePlanParams {
    anchor_year: i32,   -- current calendar year at generation time
}
```

All other generated plan content is fixed per the generate-mode specification
in [design/ux/cli.md](../ux/cli.md#generate-mode).

---

## ModelError

`ModelError` maps directly from `PlanStoreError`. Callers above the facade see
only `ModelError`; the underlying persistence type is not visible.

```
ModelError =
    | NotFound(path: Path)
    | InvalidStore(path: Path)
    | SchemaMismatch { found: SchemaVersion, expected: SchemaVersion }
    | InvalidPlan(violations: Vec<PlanViolation>)
    | Io(source: OsError)
```

---

## Construction

`Model` receives its `PlanStore` implementation at construction time — it never
selects or instantiates a backend itself. In Rust, this is expressed as a
generic over the store type (static dispatch, zero runtime cost):

```
Model<S: PlanStore> {
    store: S,
}

Model::new(store: S) -> Model<S>
```

The caller supplies the concrete type:

```
-- CLI (production):
let model = Model::new(JsonPlanStore);

-- Test harness:
let model = Model::new(MemoryPlanStore::new());
```

`Model` only ever calls `store.load()` and `store.save()` through the
`PlanStore` trait. It has no knowledge of which backend it holds beyond what
the trait exposes.

MVP data flow:

```
yarp-cli  →  Model<JsonPlanStore>  →  JsonPlanStore  →  Engine  →  stdout
```
