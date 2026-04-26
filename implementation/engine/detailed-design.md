# Engine Detailed Design

## Overview

This document is the detailed design for the projection engine. It captures
implementation-level decisions — data structures, algorithms, type signatures,
and their relationships — at a level below `design/` but above individual
source files. It is the bridge between design and implementation:
the per-file implementation plan (`plan.md`) is derived mechanically from this
spec.

The engine is the core computation pipeline that produces year-by-year financial
projections. It is the Model layer:
given a `PlanContext` (a `PlanGraph` plus precomputed caches) containing
household members, accounts, hard assets, liabilities, and assumption streams,
it validates the plan, constructs an
ephemeral projection tree from the plan's stream templates, evaluates every
stream for every year in the timeline using memoized recursive evaluation, and
returns a `Projection` containing year-indexed balance and aggregate values in
`YNV(point.year)` denomination.

For workspace layout, crate structure, dependencies, CLI integration, and build
sequence, see the [project detailed design](../detailed-design.md).

The engine lives in the `engine/` module of `yarp-core` (visibility
`pub(crate)`) and the `model.rs` facade (visibility `pub`). The engine's source
modules are:

```
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
```

Authoritative design specifications:

- [data-model.md](../../design/data-model.md) — stream primitives, entity
  schemas, procedures, UUID derivation, design invariants
- [projection.md](../../design/engine/projection.md) — three-phase projection
  algorithm
- [api.md](../../design/engine/api.md) — Model facade, Projection output,
  ModelError
- [engine principles](../../design/engine/principles.md) — MVC constraints,
  denomination conversion ownership
- [error-handling.md](../../design/error-handling.md) — error format, type
  classification, caller contract
- [implementation principles](principles.md) — pure functions, numeric precision

---

## Core Types

All entity structs, enums, `Stream`, `StreamPoint`, `PointEntry`,
`PointDenomination`, `ValueSchema`, `AttributeDef`, `ValueUnit`, `OwnedBy`,
`StreamKind`, `StreamProcedure`, `StreamStart`, `TerminationRef` must match
[data-model.md](../../design/data-model.md) field-for-field.

### PlanGraph

The `PlanGraph` struct holds all deserialized plan data in indexed collections.
It is distinct from the `Plan` entity struct -- `Plan` is the entity record
(id, name, anchor_year, household_id); `PlanGraph` is the full in-memory
aggregate containing all entities and streams. `PlanGraph` is what the store
serializes and deserializes -- it contains no derived or cached data.

```rust
struct PlanGraph {
    plan: Plan,
    household: Household,
    members: IndexMap<Uuid, Member>,
    lifecycle_events: IndexMap<Uuid, LifecycleEvent>,
    accounts: IndexMap<Uuid, Account>,
    properties: IndexMap<Uuid, Property>,
    vehicles: IndexMap<Uuid, Vehicle>,
    amortized_loans: IndexMap<Uuid, AmortizedLoan>,
    interest_only_loans: IndexMap<Uuid, InterestOnlyLoan>,
    credit_lines: IndexMap<Uuid, CreditLine>,
    streams: IndexMap<Uuid, Stream>,
    points: IndexMap<Uuid, Vec<StreamPoint>>,  // keyed by stream_id, points sorted by year
}
```

### PlanContext

`PlanContext` wraps `PlanGraph` with precomputed caches. It is what
`Model::load()` returns and what the engine operates on. The structural
separation ensures the store serializes only the graph and never touches
derived data, while the engine always has its caches available.

```rust
struct PlanContext {
    graph: PlanGraph,
    timeline_start: i32,      // min(member.birth_year) — precomputed at load time
    timeline_end: i32,        // max(member.birth_year + member.death_age) — precomputed at load time
    cpi_factors: CpiFactors,  // precomputed at load time
}
```

`Model::load()` constructs `PlanContext` by loading the `PlanGraph` via the
store, deriving the timeline bounds from member lifetimes, computing
`CpiFactors`, running validation, and wrapping the result. Timeline bounds
and CPI factors are computed once at load time and reused by `get_projection()`
— no re-derivation during the projection pipeline. Future derived caches
(e.g., entity indexes) are added here, not on `PlanGraph`.

Use `IndexMap<Uuid, T>` rather than `HashMap<Uuid, T>` for entity collections.
This gives O(1) lookup by ID with deterministic iteration order.

**Accessor methods.** `PlanGraph` exposes accessor methods that use UUID v5
derivation for stream lookup -- no scanning. Since stream IDs are deterministic
from `(owner_id, label)`, any stream can be located by computing its ID and
indexing directly:

```rust
impl PlanGraph {
    /// Look up a stream by its owner and label. Computes the deterministic
    /// UUID v5 from (owner_id, label) and indexes into the streams map.
    fn stream(&self, owner_id: Uuid, label: &str) -> Option<&Stream> {
        let id = stream_id(owner_id, label);
        self.streams.get(&id)
    }

    /// Look up stored points for a stream by owner and label.
    fn stream_points(&self, owner_id: Uuid, label: &str) -> Option<&Vec<StreamPoint>> {
        let id = stream_id(owner_id, label);
        self.points.get(&id)
    }
}
```

This eliminates all stream-scanning patterns. CPI resolution, balance template
lookup, rate stream lookup -- all become `plan_graph.stream(owner_id, label)`.

**Owner label lookup.** `PlanGraph` also exposes a method to resolve the
user-facing display name for any `OwnedBy` reference. This is the canonical
path from a stream to its owner's human-readable label -- callers never
search entity collections directly.

```rust
impl PlanGraph {
    fn owner_label(&self, owner: &OwnedBy) -> &str {
        match owner {
            OwnedBy::Member(id) => &self.members[id].given_name,
            OwnedBy::Account(id)
            | OwnedBy::MemberAccount(_, id)
            | OwnedBy::JointAccount(id) => &self.accounts[id].label,
            OwnedBy::Property(id) => &self.properties[id].label,
            OwnedBy::Vehicle(id) => &self.vehicles[id].label,
            OwnedBy::Liability(id) => {
                if let Some(l) = self.amortized_loans.get(id) {
                    &l.label
                } else if let Some(l) = self.interest_only_loans.get(id) {
                    &l.label
                } else {
                    &self.credit_lines[id].label
                }
            }
            OwnedBy::Plan(_) | OwnedBy::Relationship(_, _)
            | OwnedBy::PolicyTable(_) => panic!("no display label for {owner:?}"),
        }
    }
}
```

### Stream

```rust
struct Stream {
    id:           Uuid,
    kind:         StreamKind,
    label:        Option<String>,
    owner:        OwnedBy,
    start:        Option<StreamStart>,
    terminates:   Option<TerminationRef>,
    inputs:       IndexMap<String, Uuid>,
    procedure:    StreamProcedure,
    value_schema: ValueSchema,
}

type StreamTemplate = Stream;
```

`StreamTemplate = Stream` is a type alias defined in `types/stream.rs`
alongside the `Stream` struct. It distinguishes the role: a `StreamTemplate`
defines a computation (procedure, inputs, seed point) that produces a `Stream`
when evaluated. See [data-model.md](../../design/data-model.md#stream).

### StreamKind

```rust
enum StreamKind {
    DollarStream,
    RateStream,
    MemberLifecycleStream,
    RelationSpouseStream,
}
```

### StreamStart

```rust
enum StreamStart {
    CalendarYear(i32),
    MemberAge { member_id: Uuid, age: i32 },
}
```

### TerminationRef

```rust
enum TerminationRef {
    OnEvent { event_id: Uuid },
}
```

### StreamProcedure

```rust
enum StreamProcedure {
    Stored,
    Additive,
    Product,
    EndOfYearGrowth { cpi_stream_id: Uuid },
    AmortizedSchedule { cpi_stream_id: Uuid, periods_per_year: i32 },
    InterestOnly,
}
```

### ValueSchema and AttributeDef

```rust
struct ValueSchema {
    attributes: Vec<AttributeDef>,
}

struct AttributeDef {
    key:     String,
    unit:    ValueUnit,
    initial: Option<Decimal>,
}

enum ValueUnit {
    Decimal,
    Rate,
    Years,
    Count,
    Age,
}
```

`ValueSchema` wraps `Vec<AttributeDef>` as a named struct, not a bare Vec.

### StreamPoint and PointEntry

```rust
struct StreamPoint {
    stream_id: Uuid,
    year:      i32,
    entries:   IndexMap<String, PointEntry>,
}

struct PointEntry {
    amount:       Decimal,
    denomination: PointDenomination,
}
```

### PointDenomination

```rust
enum PointDenomination {
    Yzv,
    Pnv { ref_year: i32 },
    Ynv { target_year: i32 },
}
```

No `Cnv` variant -- CNV never appears on stored points (design invariant 2).

### OwnedBy

```rust
enum OwnedBy {
    Plan(Uuid),
    Member(Uuid),
    Account(Uuid),
    MemberAccount(Uuid, Uuid),       // (member_id, account_id)
    JointAccount(Uuid),
    Property(Uuid),
    Vehicle(Uuid),
    Liability(Uuid),
    Relationship(Uuid, Uuid),        // (from_member, to_member)
    PolicyTable(Uuid),
}
```

### Entity Structs

Implement each entity struct matching
[data-model.md](../../design/data-model.md#entity-schemas) field-for-field:

```rust
struct Plan {
    id:           Uuid,
    name:         String,
    anchor_year:  i32,
    household_id: Uuid,
}

struct Household {
    id:      Uuid,
    plan_id: Uuid,
}

struct Member {
    id:           Uuid,
    household_id: Uuid,
    given_name:   String,
    family_name:  String,
    birth_year:   i32,
    death_age:    i32,
    role:         MemberRole,
}

enum MemberRole { Primary, Spouse }

struct LifecycleEvent {
    id:        Uuid,
    plan_id:   Uuid,
    member_id: Uuid,
    kind:      EventKind,
    age:       i32,
    label:     Option<String>,
}

enum EventKind { Retirement, SocialSecurityClaiming, MedicareEligibility, Custom }

struct Account {
    id:            Uuid,
    household_id:  Uuid,
    owner:         AccountOwner,
    label:         String,
    closing_year:  Option<i32>,
    coverage_type: Option<HsaCoverageType>,
}

enum AccountOwner {
    MemberOwned(Uuid),
    JointOwned(Uuid),
}

enum HsaCoverageType { SelfOnly, Family }
enum FilingStatus { Single, MarriedFilingJointly }

struct Property {
    id:                Uuid,
    plan_id:           Uuid,
    label:             String,
    purchase_year:     i32,
    sale_year:         Option<i32>,
    value_yzv:         Decimal,
    appreciation_rate: Decimal,
}

struct Vehicle {
    id:                Uuid,
    plan_id:           Uuid,
    label:             String,
    purchase_year:     i32,
    disposal_year:     Option<i32>,
    value_yzv:         Decimal,
    depreciation_rate: Decimal,
}

struct AmortizedLoan {
    id:               Uuid,
    plan_id:          Uuid,
    label:            String,
    start_year:       i32,
    end_year:         i32,
    principal:        Decimal,
    interest_rate:    Decimal,
    payment_amount:   Decimal,
    periods_per_year: i32,
}

struct InterestOnlyLoan {
    id:            Uuid,
    plan_id:       Uuid,
    label:         String,
    start_year:    i32,
    end_year:      Option<i32>,
    balance:       Decimal,
    interest_rate: Decimal,
}

struct CreditLine {
    id:            Uuid,
    plan_id:       Uuid,
    label:         String,
    start_year:    i32,
    end_year:      Option<i32>,
    balance:       Decimal,
    credit_limit:  Decimal,
    interest_rate: Decimal,
}

```

All entity and stream types derive `Serialize`/`Deserialize` via serde.

### UUID v5 Derivation

Define `YARP_NAMESPACE` as a constant `Uuid` in `types/ids.rs`. Implement the
following derivation functions per
[data-model.md](../../design/data-model.md#uuid-derivation):

```rust
const YARP_NAMESPACE: Uuid = /* a fixed UUID constant */;

fn plan_id(name: &str) -> Uuid {
    Uuid::new_v5(&YARP_NAMESPACE, format!("plan:{name}").as_bytes())
}

fn household_id(plan_id: Uuid) -> Uuid {
    Uuid::new_v5(&YARP_NAMESPACE, format!("{plan_id}:household").as_bytes())
}

fn member_id(household_id: Uuid, given: &str, family: &str) -> Uuid {
    Uuid::new_v5(&YARP_NAMESPACE, format!("{household_id}:member:{given}.{family}").as_bytes())
}

fn event_id(member_id: Uuid, kind: &str) -> Uuid {
    Uuid::new_v5(&YARP_NAMESPACE, format!("{member_id}:event:{kind}").as_bytes())
}

fn entity_id(parent_id: Uuid, entity_type: &str, label: &str) -> Uuid {
    Uuid::new_v5(&YARP_NAMESPACE, format!("{parent_id}:{entity_type}:{label}").as_bytes())
}

fn stream_id(owner_id: Uuid, label: &str) -> Uuid {
    Uuid::new_v5(&YARP_NAMESPACE, format!("{owner_id}:{label}").as_bytes())
}

fn point_id(stream_id: Uuid, year: i32) -> Uuid {
    Uuid::new_v5(&YARP_NAMESPACE, format!("{stream_id}:{year}").as_bytes())
}
```

For composite owners (`MemberAccount`, `Relationship`), `owner_id` is the
concatenation of constituent UUIDs: `"{member_id}:{account_id}"` or
`"{from_member_id}:{to_member_id}"`.

---

## Model Facade API

`Model` is the sole public entry point into the Model layer. It is a unit
struct used as a namespace for associated functions — not an instance that
holds state. Persistence is internal: the default methods use `JsonPlanStore`;
test entry points accept any `PlanStore` implementation. Callers never see
`PlanStore`, `JsonPlanStore`, or engine internals.

```rust
struct Model;

impl Model {
    /// Load a plan from a directory using JsonPlanStore.
    fn load(dir: &Path) -> Result<PlanContext, ModelError>;

    /// Generate a minimal plan directory using JsonPlanStore.
    fn generate(dir: &Path, params: GeneratePlanParams) -> Result<(), ModelError>;

    /// Run the projection engine. Pure computation — no store needed.
    fn get_projection(plan: &PlanContext) -> Result<Projection, ModelError>;

    /// Test entry point: load with an arbitrary PlanStore backend.
    fn load_with(store: &impl PlanStore, dir: &Path) -> Result<PlanContext, ModelError>;

    /// Test entry point: generate with an arbitrary PlanStore backend.
    fn generate_with(store: &impl PlanStore, dir: &Path, params: GeneratePlanParams) -> Result<(), ModelError>;
}
```

The default `load()` and `generate()` delegate to `load_with(&JsonPlanStore, dir)`
and `generate_with(&JsonPlanStore, dir, params)` respectively. Test harnesses
call `load_with` / `generate_with` directly with `MemoryPlanStore`.

`PlanStore`, `JsonPlanStore`, and `MemoryPlanStore` remain `pub(crate)` — they
are never visible outside `yarp-core`. `ModelError` maps 1:1 from
`PlanStoreError`; `Model` converts at the facade boundary.

### Projection

```rust
struct Projection {
    years: Vec<i32>,                          // projection year range, ascending
    root_id: Uuid,                            // net-worth aggregate stream (tree walk entry point)
    streams: Vec<Stream>,                     // ephemeral projection streams
    points: IndexMap<Uuid, Vec<StreamPoint>>, // points keyed by stream id, denomination YNV(point.year)
}
```

`Projection` is the output of `get_projection()`. It contains the ephemeral
projection `Stream`s (created from `StreamTemplate`s) and the resolved
`MemberLifecycleStream` records. Callers navigate by filtering on
`stream.label` and `stream.owner`.

### GeneratePlanParams

```rust
struct GeneratePlanParams {
    anchor_year: i32,
}
```

---

## Memo Table

The memo table uses `BTreeMap<(Uuid, i32), (Decimal, PointDenomination)>` as
its backing store. Every entry records both the computed value and its
denomination. This allows denomination-aware procedures to read an input's
denomination from a single `eval()` call without a separate lookup path.

```rust
struct MemoTable {
    values: BTreeMap<(Uuid, i32), (Decimal, PointDenomination)>,
}

impl MemoTable {
    fn get(&self, stream_id: Uuid, year: i32) -> Option<(Decimal, PointDenomination)> {
        self.values.get(&(stream_id, year)).copied()
    }

    fn set(&mut self, stream_id: Uuid, year: i32, value: Decimal, denom: PointDenomination) {
        self.values.insert((stream_id, year), (value, denom));
    }

    fn get_prior_value(&self, stream_id: Uuid, year: i32) -> Option<Decimal> {
        self.values.get(&(stream_id, year - 1)).map(|(v, _)| *v)
    }
}
```

The composite key `(Uuid, i32)` sorts by UUID first, then by year within each
UUID. This provides:

- O(log n) single-key lookup. For the expected memo table size of ~8,000
  entries (100 streams x 80 years), that is ~13 comparisons -- negligible.
- Direct prior-year lookup via `get_prior_value(&(stream_id, y - 1))` for
  recurrence formulas (which need only the value, not the denomination).

The memo table is a computation cache — it exists so `eval` doesn't recompute
values that are needed multiple times (e.g., `p(y-1)` for the recurrence, or a
shared input referenced by multiple streams). It is not the output. The
`Projection` is built constructively during the sweep.

Lifecycle age values are written directly to the main memo table during Phase 3
step 1 (before standard eval) with denomination `Ynv { target_year: year }`.
No separate `LifecycleTable` is needed.

---

## CPI Cumulative Product

The CPI cumulative factor table is precomputed at plan load time and stored on
`PlanContext`. The projection sweep reads from this table -- no CPI computation
during the sweep.

The CPI cumulative factor for any year is a pure function of the plan's CPI
stream points and the anchor year. It does not depend on projection state. This
means:

- `yzv_to_ynv` is an O(1) Vec index lookup during the sweep.
- The factor table is available to any future consumer (validation,
  display-layer CNV conversion) without requiring a projection run.
- When CPI assumptions are updated, the table is recomputed once from the
  modified CPI stream. The cost is O(timeline) -- trivial.

### CpiFactors

```rust
/// Precomputed cumulative CPI factors for O(1) year-indexed lookup.
/// Backed by a Vec with an offset: index 0 = timeline_start.
/// Factor at anchor_year is always 1.0. Years before anchor have
/// factors < 1.0 (deflation); years after have factors > 1.0.
struct CpiFactors {
    timeline_start: i32,
    factors: Vec<Decimal>,  // factors[y - timeline_start] = cumulative factor for year y
}

impl CpiFactors {
    fn get(&self, year: i32) -> Decimal {
        self.factors[(year - self.timeline_start) as usize]
    }
}
```

### compute_cpi_factors

Compute during `PlanContext` construction (after the store loads the
`PlanGraph`). The table spans `timeline_start..=timeline_end`, covering the
full plan timeline. Years before `anchor_year` have factors < 1.0 (backward
deflation); `anchor_year` has factor 1.0; years after have factors > 1.0
(forward inflation).

```rust
fn compute_cpi_factors(
    cpi_stream_id: Uuid,
    points: &IndexMap<Uuid, Vec<StreamPoint>>,
    anchor_year: i32,
    timeline_start: i32,
    timeline_end: i32,
) -> CpiFactors {
    let len = (timeline_end - timeline_start + 1) as usize;
    let anchor_offset = (anchor_year - timeline_start) as usize;
    let mut factors = vec![Decimal::ZERO; len];

    // Anchor year: factor = 1.0
    factors[anchor_offset] = Decimal::ONE;

    // Forward pass: anchor_year+1 ..= timeline_end
    let mut cumulative = Decimal::ONE;
    for year in (anchor_year + 1)..=timeline_end {
        let cpi_rate = carry_forward_value(cpi_stream_id, year - 1, points);
        cumulative *= Decimal::ONE + cpi_rate;
        factors[(year - timeline_start) as usize] = cumulative;
    }

    // Backward pass: anchor_year-1 ..= timeline_start (descending)
    cumulative = Decimal::ONE;
    for year in (timeline_start..anchor_year).rev() {
        let cpi_rate = carry_forward_value(cpi_stream_id, year, points);
        cumulative /= Decimal::ONE + cpi_rate;
        factors[(year - timeline_start) as usize] = cumulative;
    }

    CpiFactors { timeline_start, factors }
}
```

`carry_forward_value` resolves the CPI rate for a given year using standard
carry-forward semantics on the CPI stream's points (most recent point at or
before the query year). This is the same resolution logic used by the `Stored`
procedure but called directly on the points vec -- no eval/memo machinery
needed.

**Backward deflation:** For years before `anchor_year`, the factor is the
reciprocal of the cumulative inflation from that year to `anchor_year`. If
CPI is 3% for all years: factor(anchor-1) = 1/1.03 ≈ 0.9709, factor(anchor-2)
= 1/(1.03)² ≈ 0.9426. This means `yzv_to_ynv($100, anchor-1)` returns ~$97.09
-- the YZV amount deflated to the earlier year's nominal dollars. This is the
correct conversion: a $100 YZV seed for a property purchased before the anchor
year should be worth less in that earlier year's nominal terms.

### yzv_to_ynv

```rust
fn yzv_to_ynv(value: Decimal, target_year: i32, cpi_factors: &CpiFactors) -> Decimal {
    if value == Decimal::ZERO { return Decimal::ZERO; }
    value * cpi_factors.get(target_year)
}
```

The zero-value fast path avoids unnecessary lookups.

---

## Carry-Forward and Stored Point Resolution

### Cursor-Based Carry-Forward

Each stream's points are stored as a sorted `Vec<StreamPoint>` (sparse --
only explicit user-set points, typically 1-5 elements per stream). During the
year sweep, each stream maintains a cursor index into its points vec. Since the
sweep advances monotonically through years, carry-forward resolution is O(1)
per year:

```
if cursor has a next point and next_point.year <= y:
    advance cursor
return current point's (value, denomination)
```

No binary search, no linear scan. The cursor advances at most once per point
over the stream's lifetime. Cursors are initialized at the start of the sweep
and are projection-local state (not stored on the plan).

### stored_point_value

`stored_point_value` returns `(Decimal, PointDenomination)` -- value and
denomination together from the cursor's current point.

```rust
fn stored_point_value(
    stream_id: Uuid,
    year: i32,
    ctx: &EvalContext,
) -> (Decimal, PointDenomination)
```

Both fields are read from the same `PointEntry`. The denomination flows
through `eval()` into the memo table, so consumers of `eval()` receive both
the value and its denomination without a separate lookup.

Carry-forward applies to ALL value units:

- `Decimal` falls back to `Decimal::ZERO`.
- `Rate`, `Count`, `Years`, `Age` fall back to `AttributeDef.initial`.

This applies to the `Stored` procedure and to seed lookups.

---

## Active Range Resolution

`resolved_start` and `resolved_end` are free functions that take `&PlanGraph`
(accessed via `plan_ctx.graph` at call sites). They are only called for streams
with `start: Some(...)` — aggregate streams (`start: None`) skip range
checking entirely. No separate `EntityIndex` -- `PlanGraph` already holds all
entity collections keyed by UUID in `IndexMap`s.

```rust
fn resolved_start(stream: &Stream, plan: &PlanGraph) -> i32;   // panics if start is None
fn resolved_end(stream: &Stream, plan: &PlanGraph, timeline_end: i32) -> i32;
```

### resolved_start

- `CalendarYear(y)` -> `y`
- `MemberAge { member_id, age }` -> `plan.members[member_id].birth_year + age`

### resolved_end

Priority chain (applied in order):

1. If `stream.terminates = OnEvent { event_id }`:
   `plan.lifecycle_events[event_id]` -> `member.birth_year + event.age`
2. Else if `owner` is `Member(id)` or `MemberAccount(member_id, _)`:
   `member.birth_year + member.death_age`
3. Else if `owner` is `Account(id)` or `JointAccount(id)`:
   `account.closing_year.unwrap_or(timeline_end)`
4. Else if `owner` is `Property(id)`:
   `property.sale_year.unwrap_or(timeline_end)`
5. Else if `owner` is `Vehicle(id)`:
   `vehicle.disposal_year.unwrap_or(timeline_end)`
6. Else if `owner` is `Liability(id)`:
   look up by ID across all three liability types; use the entity's `end_year`
   field (unwrap_or timeline_end).
7. Else (Plan-owned, Relationship, PolicyTable):
   `timeline_end`

### identity_value

Returns `(Decimal, PointDenomination)`:

- `Decimal` unit: `(Decimal::ZERO, Yzv)` -- zero converts to zero in any
  denomination; `Yzv` is a safe default.
- `Rate`, `Count`, `Years`, `Age` units: carry-forward from current cursor
  position, with its stored denomination. Fall back to
  `(AttributeDef.initial, Yzv)`.

### Aggregate Streams

Aggregate streams have `start: None` — they are derived, not concrete. They
have no lifecycle of their own. `eval` always dispatches for them (no range
check). If all children produce identity (zero) for a given year, the
`Additive` sum is zero — the aggregate naturally reflects its children's
activity without needing its own active range.

---

## Procedure Dispatch

Pure enum `match` dispatch on `StreamProcedure`. No `Procedure` trait. The
procedure set is closed (6 variants). Enum dispatch with a `match` gives
exhaustive checking at compile time and static dispatch.

### Dispatch Function Signature

```rust
fn dispatch(
    stream: &Stream,
    stream_id: Uuid,
    year: i32,
    ctx: &EvalContext,
    memo: &mut MemoTable,
) -> (Decimal, PointDenomination)
```

### EvalContext

The `EvalContext` borrows from the `ProjectionTree` and `PlanContext` for the
duration of the sweep:

```rust
struct EvalContext<'a> {
    tree: &'a ProjectionTree,
    plan_ctx: &'a PlanContext,
}
```

All fields needed during eval -- `streams`, `points`, `anchor_year`,
`cpi_factors`, `timeline_start`, `timeline_end` -- are accessible through
these two references. `tree` provides the merged stream map and projection
metadata. `plan_ctx` provides entity lookups, CPI factors, timeline bounds,
and the `stream()` accessor.

### Stored

```
(value, denom) = stored_point_value(stream_id, year, ctx)
result = (value, denom)
```

No inputs. Returns the carry-forward value and its denomination as-is.

### Additive

```
total = sum(eval(input_id, year).0 for input_id in stream.inputs.values())
result = (total, Ynv { target_year: year })
```

The sum of already-evaluated inputs. Denomination is `Ynv` because children
are either already-converted dollar values or dimensionless rates -- in either
case the sum is in the year's frame.

### Product

```
w = eval(stream.inputs["weight"], year).0
r = eval(stream.inputs["rate"], year).0
result = (w * r, Ynv { target_year: year })
```

### EndOfYearGrowth

Denomination-aware. All inputs are evaluated via `eval()`, which returns
`(Decimal, PointDenomination)`. The denomination on the returned tuple tells
the procedure whether to apply YZV-to-YNV conversion.

**Base case** (`year == resolved_start`):

```
(seed, seed_denom) = stored_point_value(stream_id, year, ctx)
if seed_denom is YZV:
    value = yzv_to_ynv(seed, year, ctx.plan_ctx.cpi_factors)
elif seed_denom is PNV:
    value = seed  // already nominal
result = (value, Ynv { target_year: year })
```

The base case uses `stored_point_value` on the stream's own seed point --
this is the stream's own stored data, not an input stream evaluation.

**Recurrence** (`year > resolved_start`):

```
prev = memo.get_prior_value(stream_id, year)
(rate, _) = eval(stream.inputs["rate"], year, ctx, memo)
net_flow = 0
for (key, input_id) in stream.inputs:
    if key == "rate": continue
    (val, denom) = eval(input_id, year, ctx, memo)
    if denom is YZV:
        net_flow += yzv_to_ynv(val, year, ctx.plan_ctx.cpi_factors)
    elif denom is PNV:
        net_flow += val
value = prev * (1 + rate) + net_flow
value = apply_balance_bound(value, stream.owner)
result = (value, Ynv { target_year: year })
```

All inputs go through `eval()`, which enforces active range and identity
semantics. A contribution stream that terminates at retirement (via `OnEvent`)
returns `(ZERO, Yzv)` for years past `resolved_end` -- zero converts to zero,
so it contributes nothing to `net_flow`. The denomination on the tuple tells
the procedure whether to apply YZV-to-YNV conversion. Rate inputs are
dimensionless; their denomination is ignored (the `_` discard).

### AmortizedSchedule

**Base case:**

```
(seed, seed_denom) = stored_point_value(stream_id, year, ctx)
result = (seed, seed_denom)  // PNV nominal principal, negative
```

**Recurrence:**

```
prev = memo.get_prior_value(stream_id, year)
(r_annual, _) = eval(stream.inputs["rate"], year, ctx, memo)
k = procedure.periods_per_year
r_p = r_annual / k
(pmt_p, _) = eval(stream.inputs["payment"], year, ctx, memo)  // nominal, no conversion
extra = 0
for (key, input_id) in stream.inputs:
    if key in ["rate", "payment"]: continue
    (val, denom) = eval(input_id, year, ctx, memo)
    if denom is YZV:
        extra += yzv_to_ynv(val, year, ctx.plan_ctx.cpi_factors)
    elif denom is PNV:
        extra += val
if r_p == 0:
    value = prev + pmt_p * k + extra
else:
    compound = pow_decimal(1 + r_p, k)  // integer exponentiation via repeated squaring
    value = prev * compound + pmt_p * (compound - 1) / r_p + extra
value = min(value, 0)  // ceiling at zero for liabilities
result = (value, Ynv { target_year: year })
```

All inputs go through `eval()`, which enforces active range and identity
semantics. A payment stream that terminates returns zero past its end. Extra
cash-flow inputs (prepayments) likewise respect their active range. The
denomination on each `eval()` return tells the procedure whether to apply
YZV-to-YNV conversion.

### InterestOnly

```
(seed, seed_denom) = stored_point_value(stream_id, resolved_start, ctx)
result = (seed, seed_denom)
```

The balance is constant for every active year. The stream emits the same
nominal opening balance for each year. The denomination passes through from
the seed (PNV) -- liability values are stored in nominal dollars, already in
YNV terms for each year.

### pow_decimal

Use `pow_decimal` with repeated squaring for integer exponents. Do NOT use
`rust_decimal`'s `powd()` which takes a Decimal exponent and uses
floating-point internally, introducing precision bugs.

```rust
fn pow_decimal(base: Decimal, exp: i32) -> Decimal {
    // Integer exponentiation via repeated squaring
    // Handles exp == 0 -> 1, negative exponents if needed
}
```

### Balance Bound

```rust
fn apply_balance_bound(value: Decimal, owner: &OwnedBy) -> Decimal {
    match owner {
        Account(_) | MemberAccount(_, _) | JointAccount(_)
        | Property(_) | Vehicle(_) => max(value, Decimal::ZERO),  // asset floor
        Liability(_) => min(value, Decimal::ZERO),                // liability ceiling
        _ => value,  // plan-owned aggregates, members, etc. -- no bound
    }
}
```

Include `MemberAccount` and `JointAccount` for defensive completeness. Even
though MVP balance streams use `Account(id)` ownership, the exhaustive match
prevents future bugs.

---

## Tree Construction

Phase 2 creates a projection `Stream` for every `StreamTemplate` in the plan --
both balance templates and aggregate templates. Plan streams that are already
real streams (rates, contributions, CPI, allocations, lifecycle) are included
by read-only reference. The result is a single merged `IndexMap<Uuid, Stream>`
that `eval` operates on. The aggregate tree structure is entirely data-driven --
the engine does not classify accounts into buckets.

### ProjectionTree

```rust
struct ProjectionTree {
    streams: IndexMap<Uuid, Stream>,          // all streams: plan streams + projection streams from templates
    points: IndexMap<Uuid, Vec<StreamPoint>>, // plan points by read-only reference
    root_id: Uuid,                            // net-worth projection stream
    lifecycle_ids: Vec<Uuid>,                 // MemberLifecycleStream IDs
    cpi_stream_id: Uuid,                      // resolved once
}
```

Timeline bounds (`timeline_start`, `timeline_end`) and `anchor_year` are read
from `PlanContext` via `EvalContext` — not duplicated on `ProjectionTree`.

### Build Steps

`build_projection_tree(plan_ctx: &PlanContext) -> ProjectionTree`

1. Reference all plan streams (rates, contributions, CPI, allocations,
   lifecycle, relation streams) and their points by read-only reference into
   the tree.
2. Resolve `cpi_stream_id` via `plan_graph.stream(plan_id, "cpi")`. Existence
   is guaranteed by `check_cpi_stream_exists` at validation time.
3. Collect `lifecycle_ids`: all streams with `kind = MemberLifecycleStream`.
4. For every `StreamTemplate` in the plan -- both balance templates and
   aggregate templates -- create a projection `Stream` with the same
   `procedure`, `inputs`, `owner`, `start`, and `terminates`. The template's
   seed `StreamPoint` remains in the points map under the template's stream
   ID -- `eval`'s base case reads it from there.
5. Set `root_id` to the projection `Stream` created from the `"net-worth"`
   aggregate template (`plan_graph.stream(plan_id, "net-worth")`).
6. Wire `cpi_stream_id` into every `EndOfYearGrowth` and `AmortizedSchedule`
   procedure variant in the tree (if not already set from plan construction).

---

## The eval() Loop

Memoized recursive evaluation. Outer year loop, inner depth-first recursion.
Lifecycle resolution precedes root eval in each year.

### Sweep Loop

```
memo = MemoTable::new()
projection = Projection {
    years: (plan_ctx.timeline_start..=plan_ctx.timeline_end).collect(),
    root_id: tree.root_id,
    streams: tree.streams.values().clone(),
    points: IndexMap::new(),
}

for year in plan_ctx.timeline_start..=plan_ctx.timeline_end:
    // Step 1: resolve all lifecycle streams
    for lifecycle_id in &tree.lifecycle_ids:
        resolve_lifecycle(lifecycle_id, year, &plan_ctx.graph, &mut memo)

    // Step 2: evaluate projection root (depth-first recursion evaluates all reachable streams)
    eval(tree.root_id, year, &ctx, &mut memo)

    // Step 3: collect output points for this year
    for stream in &tree.streams:
        (value, denom) = memo.get(stream.id, year)
        point = StreamPoint {
            stream_id: stream.id,
            year,
            entries: { stream.value_schema.attributes[0].key: PointEntry { amount: value.round_dp(2), denomination: denom } },
        }
        projection.points[stream.id].push(point)
```

Output points are built year by year as the sweep progresses. Rounding to 2
decimal places is applied here — all intermediate computation in the memo
table is full `Decimal` precision.

### eval Function

```
fn eval(stream_id: Uuid, year: i32, ctx: &EvalContext, memo: &mut MemoTable) -> (Decimal, PointDenomination):
    if let Some(cached) = memo.get(stream_id, year):
        return cached

    stream = ctx.tree.streams[stream_id]

    if stream.start.is_none():
        // Derived stream (aggregate) — no active range, always dispatch.
        (value, denom) = dispatch(stream, stream_id, year, ctx, memo)
    elif year < resolved_start(stream, &ctx.plan_ctx.graph) or year > resolved_end(stream, &ctx.plan_ctx.graph, ctx.plan_ctx.timeline_end):
        (value, denom) = identity_value(stream, year, ctx)
    else:
        (value, denom) = dispatch(stream, stream_id, year, ctx, memo)

    memo.set(stream_id, year, value, denom)
    return (value, denom)
```

Every `eval` call returns both the computed value and its denomination. For
`Stored` streams this is the denomination from the underlying `StreamPoint`
(YZV or PNV). For computed streams (`Additive`, `Product`, `EndOfYearGrowth`,
`AmortizedSchedule`) it is `Ynv { target_year: year }`. Consumers that need
denomination-aware conversion (e.g., `EndOfYearGrowth` reading a cash-flow
input) destructure the tuple and branch on the denomination.

### Cycle Detection

Add a recursion depth counter to `eval`. If depth exceeds 32, return
`ModelError::InvalidPlan` with a cycle detection violation. This is cheap
insurance against malformed plans causing stack overflow.

Additionally, add a DFS-based cycle detection check to the validation pipeline
that verifies the `inputs` graph is a DAG before projection begins.

---

## Lifecycle Stream Handling

`MemberLifecycleStream` has a single attribute (`age`), derived as
`y - birth_year`. Age is written to the main memo table as a `Decimal`. No
separate `LifecycleTable` is needed.

`MemberLifecycleStream` must be resolved before any stream that depends on
member age (design invariant 9).

### Phase 3 Step 1 Resolution

```
for each MemberLifecycleStream:
    member = plan_graph.members[stream.owner.member_id]
    age = year - member.birth_year
    memo.set(stream_id, year, Decimal::from(age), Ynv { target_year: year })
```

### Output Collection

During output collection, lifecycle streams produce `StreamPoint` records:

```
entries: {
    "age": PointEntry { amount: age, denomination: YNV(year) },
}
```

Age is derived each year, not carried forward from stored points.

---

## Denomination Handling

All dollar amounts in the plan are stored in one of two denominations:

- **YZV** -- asset and assumption dollar values (accounts, hard assets,
  contributions). These are in `anchor_year` purchasing power.
- **PNV** -- liability dollar values (principal, balance, payment, credit
  limit). These are in nominal dollars at the entity's `start_year`.

All projection output is **YNV** -- each point carries
`denomination = YNV(point.year)`.

`eval()` returns `(Decimal, PointDenomination)`. The denomination flows from
the source: `Stored` procedure streams pass through the denomination from their
`StreamPoint`; computed streams (`Additive`, `Product`, `EndOfYearGrowth`,
`AmortizedSchedule`) return `Ynv { target_year: year }`. The memo table stores
both value and denomination, so any consumer of `eval()` has the denomination
available without a separate lookup.

Denomination-aware procedures (`EndOfYearGrowth`, `AmortizedSchedule`)
destructure the `eval()` return and branch on the denomination to determine
conversion:

- `Yzv` -> `yzv_to_ynv(value, target_year, cpi_factors)`
- `Pnv { ref_year }` -> `value` (already nominal)
- `Ynv { target_year }` -> `value` (already in target year's nominal)

Rate inputs are dimensionless; their denomination is ignored (the `_` discard).

A single stream may have points in different denominations (e.g., historical PNV
prepayments and future YZV planned prepayments). `stored_point_value()` finds
the relevant point via carry-forward -- the denomination is on the same
`PointEntry` as the value. The `Stored` procedure passes both through to
`eval()`, and from there through the memo table to any consuming procedure.

Active range enforcement is automatic: a terminated stream returns
`identity_value`, which for `Decimal` unit is `(ZERO, Yzv)`. Zero converts to
zero regardless of denomination, so terminated cash-flow streams contribute
nothing to net-flow sums.

---

## Validation Pipeline

Named validator function per invariant, plus structural checks beyond the 15
design invariants. Return `Vec<PlanViolation>`. Run during `Model::load()`
after deserialization.

### Design Invariant Checks

Numbering matches
[data-model.md](../../design/data-model.md#design-invariants):

1. `check_denomination_storage` -- all asset/assumption dollar fields are YZV;
   liability fields are PNV.
2. `check_no_cnv_on_points` -- no StreamPoint has CNV denomination.
3. (runtime, not load-time) -- projection outputs are YNV(point.year).
4. `check_allocation_weights` -- for each account and year where a weight point
   exists, allocation weights sum to 1.0. Optimization: check only years where
   any weight stream has an explicit point (carry-forward guarantees stability
   between change points). This reduces cost from O(timeline_length x accounts)
   to O(distinct_weight_points x accounts).
5. `check_member_age_refs` -- every MemberAge start references a valid
   member_id.
6. `check_birth_year_immutable` -- (structural; enforced by not exposing
   mutation).
7. `check_no_raw_age_years` -- no stream record stores age-derived calendar
   years as raw start/end.
8. `check_no_explicit_ends` -- no stream has an explicit end field.
9. (runtime) -- lifecycle streams resolved before dependents.
10. `check_spousal_relationship` -- at most one spousal pair; requires exactly
    two members with Primary/Spouse roles.
11. `check_template_seed_year` -- StreamTemplate start year >= year of its seed
    StreamPoint.
12. `check_carry_forward_initial` -- every AttributeDef with carry-forward unit
    has non-None initial.
13. `check_unique_stream_points` -- (stream_id, year) unique across all points.
14. `check_unique_owner_label` -- (owner, label) unique across all streams.
15. `check_unique_entity_labels` -- entity labels globally unique within plan.

### Structural Checks

- `check_has_members` -- plan has at least one member.
- `check_has_accounts` -- plan has at least one account.
- `check_cpi_stream_exists` -- exactly one RateStream with label="cpi" owned
  by Plan.
- `check_input_references` -- every stream.inputs value references a stream
  that exists.
- `check_event_references` -- every terminates.event_id references a
  LifecycleEvent that exists.
- `check_required_input_slots` -- EndOfYearGrowth has "rate"; AmortizedSchedule
  has "rate" and "payment"; Product has "weight" and "rate"; InterestOnly has
  "rate".
- `check_no_commas_in_labels` -- commas in labels break CSV output.
- `check_effective_rate_tree` -- every account's effective rate root has at
  least one weighted-return input.
- `check_stream_dag` -- DFS-based cycle detection on the `inputs` graph.
  Verifies the graph is a DAG before projection begins.

### PlanViolation

`PlanViolation` enum: one variant per check, carrying structured data for the
violation message. `Display` produces the `error: invalid-plan: <message>`
format per [error-handling.md](../../design/error-handling.md). When multiple
violations exist, each is emitted as a separate line with the full prefix.

---

## generate() Pipeline

`Model::generate(dir, params)` delegates to
`Model::generate_with(&JsonPlanStore, dir, params)`:

```
fn generate_with(store: &impl PlanStore, dir: &Path, params: GeneratePlanParams) -> Result<(), ModelError>
```

Produces a minimal projectable plan with two members, two 401k accounts, and a
joint bank account.

### Steps

1. Create `Plan` with `anchor_year = params.anchor_year`, derive `plan_id`.
2. Create `Household`, derive `household_id`.
3. Create two `Member` entities (Alice and Bob) with default ages:
   - Alice: birth_year = anchor_year - 35, death_age = 90, role = Primary
   - Bob: birth_year = anchor_year - 33, death_age = 90, role = Spouse
4. Create `LifecycleEvent` for each member: Retirement event at age 65.
5. Create `MemberLifecycleStream` for each member.
6. Create `RelationSpouseStream` for Alice->Bob.
7. Create plan-level assumption streams:
   - CPI RateStream (label="cpi", rate=0.03)
   - 4 return rate streams (large-cap=0.08, small-cap=0.09, international=0.07,
     bonds=0.04)
   - 4 allocation weight streams (large-cap=0.40, small-cap=0.10,
     international=0.10, bonds=0.40)
8. Create `WeightedReturn` Product streams for each asset class.
9. Create two 401k `Account` entities (one per member):
   - Each with: effective rate root (Additive), balance DollarStream
     (EndOfYearGrowth), employee contribution DollarStream (Stored, $5000 YZV),
     employer contribution DollarStream (Stored, $2500 YZV). Opening
     balance = $50,000 YZV.
10. Create a joint `Bank` account with balance stream (EndOfYearGrowth, opening
    balance = $10,000 YZV). Bank's effective rate uses bonds allocation only.
11. Create all aggregate StreamTemplates (net-worth, assets, liabilities,
    retirement, health, taxable, hard-assets, lt-liabilities, st-liabilities)
    with `start: None` per the default aggregate tree structure in
    [data-model.md](../../design/data-model.md#aggregate-streamtemplates).
12. Persist via `store.save(dir, plan_graph)` (where `store` is the `PlanStore`
    passed to `generate_with`; for the public `generate()` this is `JsonPlanStore`).

---

## Error Handling

### ModelError

```rust
#[derive(Debug, thiserror::Error)]
enum ModelError {
    #[error("error: system-error: {0}")]
    NotFound(PathBuf),

    #[error("error: invalid-plan: not a valid plan store: {0}")]
    InvalidStore(PathBuf),

    #[error("error: schema-mismatch: schema version {found}; expected {expected}")]
    SchemaMismatch { found: String, expected: String },

    #[error("{}", format_violations(.0))]
    InvalidPlan(Vec<PlanViolation>),

    #[error("error: invalid-argument: {0}")]
    InvalidArgument(String),

    #[error("error: system-error: {0}")]
    Io(#[from] std::io::Error),
}
```

### format_violations

```rust
fn format_violations(violations: &[PlanViolation]) -> String {
    violations.iter()
        .map(|v| format!("error: invalid-plan: {v}"))
        .collect::<Vec<_>>()
        .join("\n")
}
```

Error format follows
[error-handling.md](../../design/error-handling.md): `<severity>: <type>: <message>`.
Callers write the `Display` output to stderr and set exit code 1. They do not
add prefixes, parse the string, or construct their own error messages.

---

CLI integration is defined in the
[project detailed design](../detailed-design.md#cli-integration).

---

## Engine Test Cases

The project-wide testing strategy (TDD, coverage requirements, golden-file
tests, PlanBuilder, rejected alternatives, rounding convention) is defined
in the [project detailed design](../detailed-design.md#testing-strategy).
This section lists only engine-specific test cases.

### Critical Branches

Branch-decision coverage (both arms exercised) is required for: procedure
dispatch, denomination conversion, carry-forward cursor advancement, balance
bounding, and active range resolution.

### Property-Based Tests (proptest)

- Additive commutativity: reordering inputs produces the same sum.
- EndOfYearGrowth monotonicity: positive rate and positive contributions produce
  increasing balances.
- AmortizedSchedule convergence: balance approaches zero over the amortization
  term.
- Balance bounds: asset balances never go negative; liability balances never go
  positive.
- CPI product identity: cumulative factor at anchor_year is 1.0.
- Allocation weight identity: weights summing to 1.0 produce a valid effective
  rate.

### Integration Test Scenarios

Each scenario below becomes a data-driven golden-file test case:

- Single account, no contributions, fixed rate
- Single account with contributions (the canonical example)
- Amortized loan payoff over 30 years
- Credit line with payments
- Interest-only loan
- Multi-entity net worth aggregation
- Lifecycle-dependent contribution termination at retirement
- Vehicle depreciation with balance floor at zero
