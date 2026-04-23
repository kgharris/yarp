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
    /// checks the schema version and validates the plan before returning.
    /// A version mismatch returns SchemaMismatch immediately — no migration
    /// is attempted in MVP. Callers receive a clean Plan or a typed error;
    /// no raw or unversioned data escapes this boundary.
    load(dir: &Path) -> Result<Plan, ModelError>

    /// Generate a minimal runnable plan and write it to a new directory.
    ///
    /// Constructs the Plan graph from GeneratePlanParams, then delegates to
    /// PlanStore::save() to persist it. The directory must not already exist.
    generate(dir: &Path, params: GeneratePlanParams) -> Result<(), ModelError>

    /// Run the projection engine over a loaded plan. Output is always YNV —
    /// no denomination parameter. The caller supplies a Plan previously
    /// returned by load(); projection is stateless and has no side effects.
    get_projection(plan: &Plan) -> Result<Projection, ModelError>
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

## Projection

`Projection` is the output of `get_projection()`. It contains ephemeral
projection streams and the resolved `MemberLifecycleStream` records from the
plan. All dollar-denominated points carry denomination `YNV(point.year)`.

```
Projection {
    years:   Vec<i32>,                         -- projection year range, ascending
    streams: Vec<Stream>,                      -- ephemeral projection streams
    points:  Map<StreamId, Vec<StreamPoint>>,  -- points keyed by stream id
}
```

`streams` contains one entry for each:
- Member lifecycle (`kind = MemberLifecycleStream`, `owner = Member(id)`) — carries
  age and phase attributes per year
- Account active in the plan timeline (`label = "balance"`, `owner = Account(id)`)
- Property or vehicle (`label = "balance"`, `owner = Property(id) | Vehicle(id)`)
- Liability (`label = "balance"`, `owner = Liability(id)`)
- Plan-level aggregate (`label = "net-worth" | "retirement" | "health" | "taxable" |
  "hard-assets" | "assets" | "liabilities" | "lt-liabilities" | "st-liabilities"`,
  `owner = Plan(id)`)

Callers navigate by filtering on `stream.label` and `stream.owner`. The CLI finds
the balance for a specific account by locating the stream with
`label = "balance"` and `owner = Account(account_id)`, then reads its `StreamPoint`
entries in year order. Building a year-keyed table from those points is a rendering
concern in the CLI — not the engine's responsibility.

`Stream`, `StreamPoint`, and the full output stream schemas are defined in the
Projection Output Streams section of [design/data-model.md](../data-model.md).

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
    | InvalidArgument(message: String)
    | Io(source: OsError)
```

Error format, type classification, message content, and the caller contract are
defined in [design/error-handling.md](../error-handling.md).

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
yarp-cli  →  Model<JsonPlanStore>  →  JsonPlanStore (load/save)
                                   →  Engine (get_projection)  →  stdout
```
