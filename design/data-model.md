# Data Model

This document defines the data model for the YARP retirement planning engine. It
is anchored to the [conceptual model](../requirements/conceptual-model.md) and
the [bedrock principles](../bedrock-principles.md). Every design decision here
traces to an `S:`, `D:`, or `R:` path in those documents.

The stream schema is defined first. All other schemas are derived from it.

---

## Notation Conventions

**`<placeholder>`** — angle-bracket tokens in stream definitions are **user-supplied parameters**, not references to named entity fields. Each occurrence is independent: two streams that both show `<employment_start_age>` share only a name chosen for readability; each stream carries its own value, set by the user when configuring that stream. A `<placeholder>` appearing inside `MemberAge { member_id: Uuid, age: <...> }` means the user supplies the age for that specific stream — it does not imply a global field on `Member` or any other entity.

---

## Stream Primitives

Every time-varying quantity in the system is a stream. This schema applies
uniformly to all streams regardless of what value type they carry.

### Stream Lifecycle

Streams are **sparsely represented**: a stream stores only its **start**. Its end
is never stored on the stream record itself — it is derived by tracing the
stream's ownership chain.

**Stream start:**

```
StreamStart =
    | CalendarYear(year: i32)
        -- Active from this calendar year.
    | MemberAge { member_id: Uuid, age: i32 }
        -- Active when the referenced member reaches this age.
        -- Resolved as member.birth_year + age at projection time.
```

**Stream end derivation (applied in priority order):**

1. If the stream declares a `terminates` field referencing a `LifecycleEvent` → ends when the referenced lifecycle event fires (see [Lifecycle Events](#lifecycle-events) below).
2. Else if the stream is owned by a member (`OwnedBy::Member` or
   `OwnedBy::MemberAccount`) → ends at `member.birth_year + member.death_age`.
3. Else if the stream is owned by a bounded entity (e.g., `Account` with a
   `closing_year`, `Property` with a `sale_year`) → ends when that entity field
   indicates, or extends to the plan timeline end if the field is `None`.
4. Else (plan-owned streams) → extends to the plan timeline end,
   which is itself derived from all member lifetimes.

**Why no explicit ends on streams:**

Storing an explicit end year or end age on a stream creates a redundant fact
that can drift out of sync with the data it is derived from. If a 401k
contribution stream stored `end_age: 65`, it would require updating whenever
the retirement age changes. With event-driven termination, there is one
authoritative fact (the `LifecycleEvent.age`), one owner (the event entity),
and every stream that references the event terminates at the same derived year
automatically.

### Lifecycle Events

A `LifecycleEvent` is a named milestone in a member's life. It serves as a
transition anchor for streams that end before the member's natural death.
The year an event fires is always derived — never stored directly.

```
LifecycleEvent {
    id:        Uuid,
    plan_id:   Uuid,
    member_id: Uuid,
    kind:      EventKind,
    age:       i32,              -- event fires at member.birth_year + age
    label:     Option<string>,   -- required for Custom events; None for named kinds
}

EventKind = Retirement | SocialSecurityClaiming | MedicareEligibility | Custom
```

A `LifecycleEvent` with `kind = Retirement` fires at `member.birth_year +
event.age`. The event is the sole authoritative source for the retirement age —
changing it updates one record, and all streams that reference the event
terminate at the new derived year automatically.

Streams that terminate at a `LifecycleEvent` set their `terminates` field to
`OnEvent { event_id }`. Streams that run until the member's natural death do
not set `terminates` at all.

```
TerminationRef =
    | OnEvent { event_id: Uuid }
        -- Stream ends when the referenced LifecycleEvent fires.
        -- Resolved as member.birth_year + event.age at projection time.
```

### Stream

```
Stream {
    id:           Uuid,
    kind:         StreamKind,
    label:        Option<string>,
    owner:        OwnedBy,
    start:        StreamStart,             -- when the stream becomes active
    terminates:   Option<TerminationRef>,  -- None = ends at natural bound (see Stream Lifecycle)
    inputs:       Map<string, Uuid>,       -- named external stream references consumed by procedure; slot names defined by procedure
    procedure:    StreamProcedure,         -- computation functor; see Stream Kind Registry
    value_schema: ValueSchema,
}

StreamTemplate = Stream   -- type alias
```

A `StreamTemplate` is structurally identical to a `Stream`. The alias
distinguishes the role: a `StreamTemplate` defines a computation (procedure,
inputs, seed point) that *produces* a `Stream` when evaluated by the projection
engine. It does not itself carry year-indexed values — it carries a seed
`StreamPoint` and the wiring that tells the engine how to project forward.
Account balance definitions, property value definitions, and liability balance
definitions are `StreamTemplate`s. Rate streams, contribution streams, and
allocation weight streams are `Stream`s — they carry stored points across
multiple years and are consumed directly during evaluation.

**Field notes:**

- `kind` — classifies the value type of this stream: monetary, dimensionless rate,
  or one of the two household marker kinds (see [Stream Kind Registry](#stream-kind-registry)
  below). Domain meaning is carried by `label`, `owner`, `inputs`, and `procedure`.
- `label` — optional human-readable name. Used for navigation (the CLI maps
  streams by label + owner) and for identification where needed (e.g., the CPI
  stream is found by `label = "cpi"`). Not a type — just a name. Streams without
  a label are valid; they are identified by position in the `inputs` tree.
- `owner` — typed discriminant pointing to the entity that owns this stream.
  The type is known from `kind` and is structurally enforced — not implied by
  convention. See [OwnedBy](#ownedby) below.
- `start` — the first year for which the stream emits real values. For years
  outside the active range, or years within it that have no explicit anchor point,
  the stream emits its identity value per [Identity Semantics](#identity-semantics)
  below. Consumers never branch on whether a stream is active — they iterate
  and aggregate what the stream emits.
- `terminates` — if set, the stream ends when the referenced `LifecycleEvent`
  fires. If `None`, the end is derived from the ownership chain per the Stream
  Lifecycle rules.
- `inputs` — named external stream references consumed by this stream's `procedure`.
  Slot names are defined by the procedure variant:
  `EndOfYearGrowth` expects slot `"rate"` (the growth rate) plus zero or more
  cash-flow slots (any key other than `"rate"` is a net-flow contribution,
  converted YZV→YNV by the procedure);
  `AmortizedSchedule` expects slots `"rate"` and `"payment"`;
  `InterestOnly` expects slot `"rate"`;
  `Product` expects slots `"weight"` and `"rate"`;
  `Additive` slots are user-defined labels;
  `Stored` has no slots.
  A missing required slot is a plan-construction error.
- `procedure` — the computation functor. Defines how `emit(y)` combines this
  stream's stored point value and its `inputs` references to produce a scalar
  for year `y`. See `StreamProcedure` in the
  [Stream Kind Registry](#stream-kind-registry) section.
- `value_schema` — the named attributes this stream's points carry. Every stream
  uses `AttributeMap`; streams with a single time-varying quantity use a single-entry
  map with key `"value"`.

```
ValueSchema = AttributeMap(attributes: [AttributeDef])

AttributeDef {
    key:     string,
    unit:    ValueUnit,
    initial: Option<Decimal>,  -- required when unit is Rate, Count, Years, or Age;
                               --   None for Decimal (identity is always 0)
}

ValueUnit = Decimal | Rate | Years | Count | Age
```

The denomination (`YZV`, `PNV`, or `YNV`) is carried on the `StreamPoint`, not
on the `ValueSchema`.

### Identity Semantics

Every stream emits a value for every year in the plan timeline. For years with
no explicit `StreamPoint` — whether before the stream's start, after its derived
end, or within gaps — the stream emits an **identity value**. The identity is
determined by the attribute's `ValueUnit`:

| Unit | Identity | Rationale |
|------|----------|-----------|
| `Decimal` | `0` | Additive identity — contributes nothing to any sum |
| `Rate` | carry-forward | Most recent `StreamPoint`; falls back to `AttributeDef.initial` |
| `Count` | carry-forward | Enumeration values hold until a new point supersedes them; falls back to `AttributeDef.initial` |
| `Years` | carry-forward | Same as `Rate` |
| `Age` | carry-forward | Same as `Rate` |

For carry-forward attributes, resolution proceeds as follows: take the most
recent `StreamPoint` at or before the query year; if none exists, use
`AttributeDef.initial`. The `initial` value is part of the stream definition —
it is not tied to a year and applies implicitly from the stream's `start` onward.
Carry-forward resolution never fails: the `initial` value is always the terminal
fallback.

### OwnedBy

`OwnedBy` is a typed discriminant — not a polymorphic opaque UUID. This keeps
references navigable without implicit conventions. Any query that navigates from
a stream to its owner must do so without knowing the owner table from context alone.

```
OwnedBy =
    | Plan(plan_id: Uuid)
    | Member(member_id: Uuid)
    | Account(account_id: Uuid)
    | MemberAccount(member_id: Uuid, account_id: Uuid)
    | JointAccount(account_id: Uuid)
    | Property(property_id: Uuid)
    | Vehicle(vehicle_id: Uuid)
    | Liability(liability_id: Uuid)
    | Relationship(from_member: Uuid, to_member: Uuid)
    | PolicyTable(table_id: Uuid)
```

The `PolicyTable` variant is used by structured table streams (federal tax
bracket tables).

### StreamPoint

A `StreamPoint` is an explicit anchor for a specific year. Stored points
represent user-entered values, historical actuals, or projection outputs.
Years with no stored point produce the stream's identity value per
[Identity Semantics](#identity-semantics).

```
StreamPoint {
    stream_id: Uuid,                       -- references Stream.id
    year:      i32,                        -- calendar year this anchor covers
    entries:   Map<string, PointEntry>,    -- keys match AttributeDef.key from Stream.value_schema
}

PointEntry {
    amount:       Decimal,
    denomination: PointDenomination,
}

PointDenomination =
    | YZV
    | PNV(ref_year: i32)
    | YNV(target_year: i32)   -- projection output points only
```

**Invariant:** `CNV` never appears on a stored `StreamPoint`.

**Denomination on output points:** For projection output streams, each point's
denomination is `YNV(point.year)`. The denomination of any stored point is
determinable from the point record alone.

### Stream Procedures

Every stream carries a `procedure` that defines its `emit(y)` behavior. This
is the computation functor — the rule by which the stream combines its stored
point value and its `inputs` references to produce a scalar output for year `y`.

In Rust, `StreamProcedure` is a trait with a single method:

```rust
trait Procedure {
    fn combine(&self, stored: Decimal, inputs: &HashMap<String, Decimal>) -> Decimal;
}
```

Each variant below is a struct implementing this trait:

```
StreamProcedure =
    | Stored
        -- combine(stored, {}) = stored
        -- Emits its own stored StreamPoint value for year y (carry-forward or
        --   additive identity). Leaf node. inputs is empty.

    | Additive
        -- combine(stored, inputs) = sum(values(inputs))
        -- Emits the sum of all named input stream values. Slot names are
        --   user-defined labels (e.g. "assets", "liabilities", "401k",
        --   "large-cap"). stored is unused.

    | Product
        -- combine(stored, {"weight": w, "rate": r}) = w × r
        -- Emits the product of inputs["weight"] and inputs["rate"]. Used by
        --   WeightedReturn RateStreams to compute allocation_weight × return_rate.

    | EndOfYearGrowth { cpi_stream_id: Uuid }
        -- combine(stored, inputs) =
        --   p(y) = p(y-1) × (1 + r) + net_flow(y)
        -- Used by accounts, hard assets, and credit lines.
        --   inputs["rate"] is the growth rate stream (e.g. effective rate RateStream).
        --   net_flow(y) is the sum of all non-"rate" input values for year y,
        --   each converted based on its stored denomination: YZV inputs are
        --   converted to YNV using cpi_stream_id; PNV inputs are used directly.
        --   Any input key other than "rate" is a cash-flow contribution.
        --   p(y-1) is this stream's own emitted value for the prior year.
        --
        --   cpi_stream_id references the plan's designated inflation RateStream.
        --   In MVP this is the single plan-owned CPI RateStream (label="cpi"; owned by
        --   Plan). It is resolved once during projection tree construction (Phase 2)
        --   and stored here so the procedure is self-contained at evaluation time.
        --   Substituting a different inflation stream (e.g. CPI-W) requires only
        --   setting a different cpi_stream_id — the procedure is unchanged.

    | AmortizedSchedule { cpi_stream_id: Uuid, periods_per_year: i32 }
        -- combine(stored, inputs) =
        --   r_p       = inputs["rate"] / periods_per_year
        --   k         = periods_per_year
        --   pmt_p     = inputs["payment"]
        --   extra     = sum(v for key, v in inputs if key not in {"rate", "payment"})
        --   if r_p = 0:
        --       balance(y) = balance(y-1) + pmt_p × k + extra
        --   else:
        --       balance(y) = balance(y-1) × (1 + r_p)^k
        --                    + pmt_p × ((1 + r_p)^k − 1) / r_p
        --                    + extra
        --
        -- Year-over-year recurrence: applies k payment periods of standard
        -- amortization to the prior year-end balance, then adds any extra
        -- cash flows (prepayments, extra principal).
        --
        -- Base case: balance(resolved_start) = seed (nominal principal, PNV).
        -- No YZV→YNV conversion — liability seeds are nominal dollars.
        -- balance(y-1) is this stream's own emitted value for the prior year.
        --
        -- inputs["rate"] is the annual interest rate stream (divided by
        --   periods_per_year internally).
        -- inputs["payment"] is a Stored DollarStream carrying the per-period payment.
        -- Any input key other than "rate" or "payment" is a cash-flow
        --   (prepayment, extra principal), converted denomination-aware per input:
        --   YZV inputs converted to YNV using cpi_stream_id; PNV inputs used directly.
        --
        -- periods_per_year: 12=monthly, 26=biweekly, 52=weekly, 4=quarterly,
        --   1=annual. Equivalent to Excel's nper-per-year parameter.
        -- cpi_stream_id: inflation reference for YZV→YNV conversion (same as
        --   EndOfYearGrowth).
        --
        -- Because the recurrence uses the prior balance, mid-life modifications
        -- are handled naturally: rate changes, payment changes (re-amortization),
        -- and prepayments all flow through the input streams and apply to the
        -- current balance going forward.
        --
        -- **Payment denomination:** the per-period payment on a fixed-rate loan
        -- is contractually fixed in nominal dollars — it does not inflate. The
        -- procedure uses inputs["payment"] as a nominal value (no YZV→YNV
        -- conversion). Extra cash-flow inputs (prepayments) ARE converted
        -- YZV→YNV, since they represent user-planned actions expressed in
        -- today's purchasing power.

    | InterestOnly
        -- combine(stored, {"rate": r}) =
        --   balance(y) = balance(y-1)  [principal constant]
        --   interest(y) = balance(y-1) × r
        --   The balance does not change year-over-year; only interest accrues.
        --   inputs["rate"] is the interest rate stream.
        --   Equivalent to a simple interest calculation in Excel.
```

### Timeline Derivation

The plan timeline is not a configured input. It is derived from all member
lifetimes in the plan:

```
timeline_start = min(member.birth_year for all members in plan)

timeline_end   = max(
    member.birth_year + member.death_age
    for all members in plan
)
```

Member-owned streams extend to their owner's `birth_year + death_age` by
default. Streams with a `terminates` reference end at the event's derived year.
Plan-level streams extend to `timeline_end`.

A stream's resolved start is:
    CalendarYear(y)         → y
    MemberAge(mid, age)     → member(mid).birth_year + age

A stream's resolved end is determined by the derivation rules in
[Stream Lifecycle](#stream-lifecycle) above — never by a field on the stream
record.

Per [D:streams / timeline](../requirements/conceptual-model.md#d-streams),
the timeline is not independently configured. Any entity that requires an
explicit start/end year (e.g., a projection request) is a filter on the
engine-derived timeline, not the timeline's definition.

---

## Stream Kind Registry

`StreamKind` classifies every stream by its value type. Domain meaning is carried
by `label`, `owner`, `inputs`, and `procedure` — not by the kind.

```
StreamKind =
    | DollarStream          -- monetary values; stored YZV, projected YNV
    | RateStream            -- dimensionless; rates, weights, fractions
    | MemberLifecycleStream -- member age attribute (non-monetary, non-rate)
    | RelationSpouseStream  -- spousal relationship marker
```

All entity balance templates (accounts, hard assets, liabilities) are `StreamTemplate` (alias for `Stream`).
All assumption streams (CPI, return rates, allocation weights) are `RateStream`.
Projection output streams are `DollarStream` (ephemeral; never persisted).

```
AccountKind =
    | Traditional401k
    | Roth401k
    | TraditionalIra
    | RothIra
    | Hsa
    | Bank
    | Brokerage
    | JointBank
    | JointBrokerage

AccountOwner =
    | MemberOwned(member_id: Uuid)
    | JointOwned(plan_id: Uuid)

FilingStatus = Single | MarriedFilingJointly

MemberRole = Primary | Spouse

PropertyKind = PrimaryResidence | VacationHome | LandParcel

LiabilityKind = AmortizedLoan | InterestOnlyLoan | CreditLine

HsaCoverageType = SelfOnly | Family
```

**Note on `AccountKind`:** The `DollarStream` kind is shared by all account types.
The account type is carried on the `Account` entity. The engine dispatches on
`Account.account_kind`, not on a stream-kind variant per account type. Similarly,
`LiabilityKind` on the liability entity, and `PropertyKind` on the `Property`
entity, provide type discrimination without encoding it in `StreamKind`.

### Stream Label Conventions

Stream labels are plain strings — not an enum. New labels can be introduced
without schema changes. The following labels are used by convention in MVP:

| Label | Kind | Used on |
|-------|------|---------|
| `"balance"` | `DollarStream` | Balance `StreamTemplate` (plan-side) or projection output `Stream` for an account, property, vehicle, or liability |
| `"cpi"` | `RateStream` | Plan-level general inflation rate; universal YZV→YNV reference |
| `"return.<class>"` | `RateStream` | Return rate for an asset class (e.g., `"return.large-cap"`, `"return.bonds"`) |
| `"weight.<class>"` | `RateStream` | Allocation weight for an asset class (e.g., `"weight.large-cap"`) |
| `"weighted-return.<class>"` | `RateStream` | Product of weight × return for an asset class |
| `"effective-rate"` | `RateStream` | Additive root of weighted returns for an account |
| `"net-worth"` | `DollarStream` | Projection root |
| `"assets"` | `DollarStream` | Projection total assets aggregate |
| `"liabilities"` | `DollarStream` | Projection total liabilities aggregate |
| `"retirement"` | `DollarStream` | Projection retirement aggregate |
| `"health"` | `DollarStream` | Projection health aggregate |
| `"taxable"` | `DollarStream` | Projection taxable aggregate |
| `"hard-assets"` | `DollarStream` | Projection hard assets aggregate |
| `"lt-liabilities"` | `DollarStream` | Projection long-term liabilities aggregate |
| `"st-liabilities"` | `DollarStream` | Projection short-term liabilities aggregate |

Contribution streams carry labels matching their `inputs` key on the parent
balance stream (e.g., `"employee-401k"`, `"employer-401k"`). The label
serves both as the stream's identifier and as the key in the parent's
`inputs` map.

---

## Stream Composition Coverage

This section shows how the two primitive kinds — `DollarStream` and `RateStream`
— and the four procedures cover every domain stream type in the requirements. No
domain concept requires a new `StreamKind`; all variation is expressed through
`label`, `procedure`, and `inputs`.

| Domain concept | Kind | Procedure | inputs |
|---|---|---|---|
| Account balance (template) | `StreamTemplate` | `EndOfYearGrowth` | `{"rate": effective_rate_stream, "<label>": cash_flow_stream, ...}` |
| Hard asset value (template) | `StreamTemplate` | `EndOfYearGrowth` | `{"rate": rate_stream, "<label>": capital_improvement_stream, ...}` |
| Credit line balance (template) | `StreamTemplate` | `EndOfYearGrowth` | `{"rate": interest_rate_stream, "<label>": payment_stream, ...}` |
| Amortized loan balance (template) | `StreamTemplate` | `AmortizedSchedule` | `{"rate": interest_rate_stream, "payment": payment_stream, "<label>": prepayment_stream, ...}` |
| Interest-only loan balance (template) | `StreamTemplate` | `InterestOnly` | `{"rate": interest_rate_stream}` |
| Cash flow (contribution, payment, withdrawal) | `DollarStream` | `Stored` | `{}` |
| Aggregate rollup (template) | `StreamTemplate` | `Additive` | `{"<label>": child_template_or_stream, ...}` |
| Projection output leaf (balance, value) | `DollarStream` | `EndOfYearGrowth` / `AmortizedSchedule` / `InterestOnly` | same as plan template |
| Projection aggregate | `DollarStream` | `Additive` | mirrors plan aggregate template |
| CPI / inflation rate | `RateStream` | `Stored` | `{}` |
| Return rate for one asset class | `RateStream` | `Stored` | `{}` |
| Allocation weight for one asset class | `RateStream` | `Stored` | `{}` |
| Weighted return (weight × rate for one asset class) | `RateStream` | `Product` | `{"weight": allocation_weight, "rate": return_rate}` |
| Effective rate for an account | `RateStream` | `Additive` | `{"<label>": weighted_return_stream, ...}` |
| Fixed interest rate (liability-owned) | `RateStream` | `Stored` | `{}` |
| Member lifecycle (age) | `MemberLifecycleStream` | `Stored` | `{}` |
| Spousal relationship | `RelationSpouseStream` | `Stored` | `{}` |

**How the growth procedures cover all balance-like domain types:**

Every domain concept that projects a monetary value forward uses one of three
procedures, chosen by which Excel formula would be correct. Formulas are defined
authoritatively in the `StreamProcedure` enum above; the wiring differs by domain
concept as follows.

**Sign convention:** a `DollarStream` balance is positive when the entity is an asset
and negative when the entity is a liability. A cash-flow child stream carries a
positive value when it increases the balance (contribution into an account, payment
into a loan) and a negative value when it decreases it (withdrawal from an account).
The procedure determines the formula; the sign of the seed and child values determine
the direction.

The domain difference is entirely in wiring:
- An account balance template (positive) uses `EndOfYearGrowth`; `inputs["rate"]` is the
  effective-rate root; all other inputs are cash-flow `DollarStream`s.
- A property or vehicle value template (positive) uses `EndOfYearGrowth`; `inputs["rate"]`
  is a return/depreciation `RateStream`; all other inputs are capital improvements.
- A credit line template (negative) uses `EndOfYearGrowth`; `inputs["rate"]` is the interest
  rate; all other inputs are payment cash flows.
- An amortized loan template (negative) uses `AmortizedSchedule`; `inputs["rate"]` and
  `inputs["payment"]` determine the base schedule; other inputs are prepayments
  or extra principal.
- An interest-only loan template (negative) uses `InterestOnly`; `inputs["rate"]` is the
  interest rate; no other inputs.

**Rate wiring is uniform.** Every `DollarStream` that uses a growth procedure
references its rate via `inputs["rate"]`. That rate is always a `RateStream`
with `procedure = Stored`. There is no special construct per domain type — the
only variation is ownership and what the rate represents in context:

| Domain type | `inputs["rate"]` points to | Owner | Source |
|---|---|---|---|
| Account balance | effective rate root (`Additive`) | `Account(id)` | Composed from plan-level return rates and allocation weights |
| Property value | appreciation rate | `Property(id)` | Entity's `appreciation_rate` scalar → single-point `Stored` `RateStream` |
| Vehicle value | depreciation rate | `Vehicle(id)` | Entity's `depreciation_rate` scalar → single-point `Stored` `RateStream` (negative value) |
| Amortized loan | interest rate | `Liability(id)` | Entity's `interest_rate` scalar → single-point `Stored` `RateStream` |
| Interest-only loan | interest rate | `Liability(id)` | Entity's `interest_rate` scalar → single-point `Stored` `RateStream` |
| Credit line | interest rate | `Liability(id)` | Entity's `interest_rate` scalar → single-point `Stored` `RateStream` |

For entities that carry a rate scalar (all three liability types, properties,
vehicles), the scalar is realized at plan construction as a `Stored`
`RateStream` owned by the entity, with a single `StreamPoint` at `start_year`.
The `DollarStream`'s `inputs["rate"]` references this stream by ID. Variable
rates are modeled by adding additional `StreamPoint`s to the same stream —
no structural change.

**How the effective rate tree is open-ended:**

A user adds an asset class by creating a `ReturnRate` and an `AllocationWeight`
`RateStream` pair, then adding a new `WeightedReturn` entry to the account's
effective rate root's `inputs`. No schema change. No new `StreamKind`. No new
No new schema types are required — though a label may be set for display purposes.

**Future stream types:**

Income streams (wages, Social Security, pension), expense streams (living, housing,
healthcare), and withdrawal streams are all `DollarStream` children or parents,
distinguished by label. A variable-rate loan uses a `Stored` RateStream whose points
the user updates over time — structurally identical to a fixed-rate loan, just with
more points. Nothing in the requirements calls for a third `StreamKind`.

---

## Entity Schemas

Each entity below is an anchor entity — it holds configuration fields that do
not vary by year. Time-varying behavior is always delegated to streams. Every
entity has one or more streams; streams reference their owner entity by `id`.

### Plan

```
Plan {
    id:          Uuid,
    name:        string,
    anchor_year: i32,         -- defines what "year zero" means for YZV storage
    household_id: Uuid,       -- reference to Household
}
```

`anchor_year` is the denominator for all `YZV` values stored in this plan. It
does not change once set.

### Household

```
Household {
    id:      Uuid,
    plan_id: Uuid,
}
```

The household is the container for members. The household itself is not a
stream — it does not have a lifecycle. Its members do.

### Member

A member is modeled as a stream. The `Member` entity holds configuration that
does not vary by year. The `MemberLifecycleStream` stream carries the member's active
range and encodes life-stage transitions via attributes.

```
Member {
    id:             Uuid,
    household_id:   Uuid,
    given_name:     string,
    family_name:    string,
    birth_year:     i32,             -- sole calendar anchor; all ages resolve through this
    death_age:      i32,             -- planning horizon end, expressed as age
    role:           MemberRole,
}
```

`birth_year` is the member's single calendar anchor. Every derived calendar
event is computed as `birth_year + age` via `LifecycleEvent`. Storing ages
rather than calendar years means that correcting `birth_year` automatically
shifts all downstream events. Retirement age is not a member field — it is a
`LifecycleEvent` with `kind = Retirement`, the same mechanism used for Social
Security claiming, Medicare eligibility, and custom milestones.

No age-derived calendar year (i.e., `member.birth_year + age`) may be stored as
a raw `start_year` or `end_year` on a stream record. Age-relative starts are
expressed as `StreamStart::MemberAge`. Age-relative ends are expressed via
`TerminationRef::OnEvent`, where the event's `age` field is the authoritative
value.

**MemberLifecycleStream:**

```
MemberLifecycleStream for member M:
    kind         = MemberLifecycleStream
    owner        = Member(M.id)
    start        = MemberAge(M.id, age: 0)
    -- no terminates — MemberLifecycleStream ends at member death, the natural bound
    --   derived from Member.death_age
    value_schema = AttributeMap([
        { key: "age",   unit: Age   },   -- derived as y - birth_year each year (not carry-forward)
    ])
```

The `age` attribute is a special case: it is computed as `y - member.birth_year`
during Phase 3 step 1 resolution, not via the `Stored` procedure's carry-forward
rule.

**Design invariant — resolution order:** `MemberLifecycleStream` streams are resolved
before all streams that reference them. This is a named engine contract: the
engine's projection pass resolves household lifecycle streams first in each year
before evaluating any stream that depends on a member's age attribute.

**Member relation streams:**

```
RelationSpouseStream:
    kind   = RelationSpouseStream
    owner  = Relationship(from_member: Uuid, to_member: Uuid)
    start  = MemberAge(from_member.id, age: age when relationship becomes active)
    -- no terminates — a spousal relation ends when either member dies,
    --   derived from their natural bounds
```

---

## Assets — Accounts

### Account

```
Account {
    id:              Uuid,
    household_id:    Uuid,
    account_kind:    AccountKind,
    owner:           AccountOwner,
    label:           string,         -- user-assigned label (e.g., "Fidelity 401k")
    closing_year:    Option<i32>,
    coverage_type:   Option<HsaCoverageType>,  -- required when account_kind = Hsa; None for all other account kinds
}
```

### Account Balance StreamTemplate

Each account has exactly one `StreamTemplate` with `label = "balance"`. This
template defines the computation for projecting the account's balance forward.
It carries a seed `StreamPoint` (the opening balance in YZV) and wires together
the rate and contribution streams that feed the projection.

```
StreamTemplate for account A (balance):
    kind         = DollarStream
    label        = "balance"
    owner        = Account(A.id)
    start        = CalendarYear(<start_year>)
    -- end derived from Account.closing_year per stream lifecycle rule 3
    procedure    = EndOfYearGrowth
    inputs       = { "rate": effective_rate_stream_id, "<label>": contribution_stream_id, ... }
    -- "rate" is the account's effective-rate RateStream root.
    -- All other inputs are contribution DollarStreams for this account (one per active contribution type).
    -- EndOfYearGrowth treats every non-"rate" input as a net-flow contribution.
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
```

**Denomination separation:** The `StreamTemplate` holds the user-entered seed
balance (YZV or PNV depending on the entity type). The projection engine reads
this seed, converts it to `YNV(resolved_start)` if needed, and produces an
ephemeral projection `Stream` with year-over-year balances in `YNV`. The
template and the projection output are distinct — a single record does not carry
two denominations.

---

## Assets — Hard Assets

Hard assets are non-liquid stores of value. They are not part of the liquid
retirement pool but contribute to net worth.

### Property

```
Property {
    id:               Uuid,
    plan_id:          Uuid,
    property_kind:    PropertyKind,
    label:            string,
    purchase_year:    i32,
    sale_year:        Option<i32>,
    value_yzv:        Decimal,      -- [YZV] estimated value at purchase_year
    appreciation_rate: Decimal,     -- dimensionless; annual appreciation rate
}
```

```
RateStream for property P (appreciation rate):
    kind         = RateStream
    owner        = Property(P.id)
    start        = CalendarYear(P.purchase_year)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Rate }])
    -- Single StreamPoint at purchase_year with Property.appreciation_rate.
    -- Additional points model rate changes over time.

StreamTemplate for property P (value):
    kind         = DollarStream
    label        = "balance"
    owner        = Property(P.id)
    start        = CalendarYear(P.purchase_year)
    -- end derived from Property.sale_year per stream lifecycle rule 3
    procedure    = EndOfYearGrowth
    inputs       = { "rate": appreciation_rate_stream_id }
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] seed; projected YNV per year by engine
```

### Vehicle

```
Vehicle {
    id:                Uuid,
    plan_id:           Uuid,
    label:             string,
    purchase_year:     i32,
    disposal_year:     Option<i32>,
    value_yzv:         Decimal,      -- [YZV] estimated value at purchase_year
    depreciation_rate: Decimal,      -- dimensionless; annual depreciation rate (negative, e.g. -0.15)
}
```

```
RateStream for vehicle V (depreciation rate):
    kind         = RateStream
    owner        = Vehicle(V.id)
    start        = CalendarYear(V.purchase_year)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Rate }])
    -- Single StreamPoint at purchase_year with Vehicle.depreciation_rate.
    -- Additional points model rate changes over time.

StreamTemplate for vehicle V (value):
    kind         = DollarStream
    label        = "balance"
    owner        = Vehicle(V.id)
    start        = CalendarYear(V.purchase_year)
    -- end derived from Vehicle.disposal_year per stream lifecycle rule 3
    procedure    = EndOfYearGrowth
    inputs       = { "rate": depreciation_rate_stream_id }
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] seed; projected YNV per year
```

---

## Contributions

Contribution streams are `Stored` `DollarStream`s referenced by name in the
account balance stream's `inputs` map. The `EndOfYearGrowth` procedure treats
every input key other than `"rate"` as a net-flow contribution. The engine
resolves contributions for an account by reading the non-`"rate"` entries from
the account balance stream's `inputs`.

The contribution type is identified by its `inputs` key on the parent balance
stream (e.g., `"employee-401k"`, `"employer-401k"`). When contribution limits
are in scope (FUT), the engine dispatches to the appropriate limit bucket by
reading the `inputs` key. All stored values are `[YZV]`; positive = inflow.

### 401k Contributions

```
DollarStream for member M on account A (Employee401k):
    kind         = DollarStream
    label        = "employee-401k"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(retirement_event_id)
    -- retirement_event_id references LifecycleEvent { kind: Retirement }
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

DollarStream for member M on account A (Employer401k):
    kind         = DollarStream
    label        = "employer-401k"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(retirement_event_id)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

DollarStream for member M on account A (Catchup401k):
    kind         = DollarStream
    label        = "catchup-401k"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: 50)
    terminates   = OnEvent(retirement_event_id)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- When limits are in scope: age-tiered limit applies (ages 50–59 and 64+: standard
    --   catchup; ages 60–63: SECURE 2.0 super-catchup).
```

### Roth 401k Contributions

```
DollarStream for member M on account A (EmployeeRoth401k):
    kind         = DollarStream
    label        = "employee-roth-401k"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(retirement_event_id)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

DollarStream for member M on account A (EmployerRoth401k):
    kind         = DollarStream
    label        = "employer-roth-401k"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(retirement_event_id)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

DollarStream for member M on account A (CatchupRoth401k):
    kind         = DollarStream
    label        = "catchup-roth-401k"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: 50)
    terminates   = OnEvent(retirement_event_id)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- Combined catchup limit shared with Catchup401k per member when limits are in scope.
```

### IRA Contributions

```
DollarStream for member M on account A (StandardIra):
    kind         = DollarStream
    label        = "standard-ira"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: 0)
    terminates   = None
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

DollarStream for member M on account A (CatchupIra):
    kind         = DollarStream
    label        = "catchup-ira"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: 50)
    terminates   = None
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
```

### Roth IRA Contributions

```
DollarStream for member M on account A (StandardRothIra):
    kind         = DollarStream
    label        = "standard-roth-ira"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: 0)
    terminates   = None
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

DollarStream for member M on account A (CatchupRothIra):
    kind         = DollarStream
    label        = "catchup-roth-ira"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: 50)
    terminates   = None
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- Combined IRA catchup limit shared with CatchupIra per member when limits are in scope.
```

### HSA Contributions

```
DollarStream for member M on account A (EmployeeHsa):
    kind         = DollarStream
    label        = "employee-hsa"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(medicare_event_id)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

DollarStream for member M on account A (EmployerHsa):
    kind         = DollarStream
    label        = "employer-hsa"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(medicare_event_id)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

DollarStream for member M on account A (CatchupHsa):
    kind         = DollarStream
    label        = "catchup-hsa"
    owner        = MemberAccount(M.id, A.id)
    start        = MemberAge(M.id, age: 55)
    terminates   = OnEvent(medicare_event_id)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
```

### Bank and Brokerage Contributions

```
DollarStream (Bank or Brokerage):
    kind         = DollarStream
    label        = "bank" | "brokerage"
    owner        = MemberAccount(M.id, A.id) | JointAccount(A.id)
    start        = CalendarYear(<start_year>)
    terminates   = None
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
```

---

## Liabilities

**Denomination convention:** all liability dollar amounts — principal, balance,
payment, credit limit — are stored in **nominal dollars at the entity's
`start_year`**. This is an exception to the general "storage is YZV" rule.
Loan parameters are contractual terms denominated in the contract's own
currency (nominal dollars at signing). Storing them in YZV and converting back
would introduce an unnecessary conversion step that is both a bug surface and
semantically wrong — a $300,000 mortgage is $300,000 regardless of which year
is the plan's anchor. Stream seed points for liabilities carry denomination
`PNV(start_year)`. The projection formulas use these values directly without
YZV→YNV conversion.

### Amortized Loan

```
AmortizedLoan {
    id:               Uuid,
    plan_id:          Uuid,
    label:            string,
    start_year:       i32,
    end_year:         i32,             -- scheduled payoff year
    principal:        Decimal,         -- nominal opening principal at start_year
    interest_rate:    Decimal,         -- dimensionless; annual rate
    payment_amount:   Decimal,         -- nominal per-period payment amount
    periods_per_year: i32,             -- payment frequency: 12=monthly, 26=biweekly, 4=quarterly, 1=annual
}
```

```
RateStream for AmortizedLoan (interest rate):
    kind         = RateStream
    owner        = Liability(AmortizedLoan.id)
    start        = CalendarYear(start_year)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Rate }])
    -- Single StreamPoint at start_year with AmortizedLoan.interest_rate.
    -- Additional points model rate changes (ARM, refinance).

StreamTemplate for AmortizedLoan (balance):
    kind         = DollarStream
    label        = "balance"
    owner        = Liability(AmortizedLoan.id)
    start        = CalendarYear(start_year)
    -- end derived from AmortizedLoan.end_year per stream lifecycle rule 3
    procedure    = AmortizedSchedule { cpi_stream_id, periods_per_year: AmortizedLoan.periods_per_year }
    inputs       = { "rate": interest_rate_stream_id, "payment": payment_stream_id, "<label>": prepayment_stream_id, ... }
    --   payment_stream_id: a Stored DollarStream carrying AmortizedLoan.payment_amount (nominal)
    --   seed value is negative (nominal principal); recurrence derives remaining balance per year
    --   Any non-"rate"/non-"payment" inputs are extra cash flows (prepayments, extra principal)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- outstanding principal; negative
```

### Interest-Only Loan

```
InterestOnlyLoan {
    id:            Uuid,
    plan_id:       Uuid,
    label:         string,
    start_year:    i32,
    end_year:      Option<i32>,
    balance:       Decimal,    -- nominal outstanding balance at start_year
    interest_rate: Decimal,    -- dimensionless; annual rate
}
```

```
RateStream for InterestOnlyLoan (interest rate):
    kind         = RateStream
    owner        = Liability(InterestOnlyLoan.id)
    start        = CalendarYear(start_year)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Rate }])
    -- Single StreamPoint at start_year with InterestOnlyLoan.interest_rate.

StreamTemplate for InterestOnlyLoan (balance):
    kind         = DollarStream
    label        = "balance"
    owner        = Liability(InterestOnlyLoan.id)
    start        = CalendarYear(start_year)
    -- end derived from InterestOnlyLoan.end_year per stream lifecycle rule 3
    procedure    = InterestOnly
    inputs       = { "rate": interest_rate_stream_id }
    --   seed value is negative (nominal balance); emitted as-is for all active years
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- outstanding balance; negative
```

### Credit Line

```
CreditLine {
    id:            Uuid,
    plan_id:       Uuid,
    label:         string,
    start_year:    i32,
    end_year:      Option<i32>,
    balance:       Decimal,    -- nominal outstanding balance at start_year; negative (liability)
    credit_limit:  Decimal,    -- nominal credit limit at start_year
    interest_rate: Decimal,    -- dimensionless
}
```

```
RateStream for CreditLine (interest rate):
    kind         = RateStream
    owner        = Liability(CreditLine.id)
    start        = CalendarYear(start_year)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Rate }])
    -- Single StreamPoint at start_year with CreditLine.interest_rate.

StreamTemplate for CreditLine (balance):
    kind         = DollarStream
    label        = "balance"
    owner        = Liability(CreditLine.id)
    start        = CalendarYear(start_year)
    -- end derived from CreditLine.end_year per stream lifecycle rule 3
    procedure    = EndOfYearGrowth
    inputs       = { "rate": interest_rate_stream_id, "<label>": payment_stream_id, ... }
    --   seed value is negative (nominal balance); non-"rate" inputs are payment streams (positive values reduce the balance)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- revolving balance; negative
```

---

## Assumption Streams

User-configurable assumption streams are `RateStream` with `procedure = Stored`.
They are owned by the plan and use carry-forward semantics — the most recent stored
point at or before a given year is the effective value for that year.

### CPI Stream

```
RateStream (CPI):
    kind         = RateStream
    label        = "cpi"
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Rate }])   -- e.g. 0.03
```

The CPI stream is the universal inflation reference. Every `EndOfYearGrowth`
`DollarStream` uses it as the built-in CPI reference for YZV→YNV conversion of
net-flow inputs.

### Return Rate and Allocation Weight Streams

Return rates and allocation weights are independent `Stored` RateStreams owned by
the plan. Asset classes are not a closed enum — a user adds an asset class by
creating a new pair of streams.

```
RateStream (return rate for one asset class, e.g. "Large Cap Equities"):
    kind         = RateStream
    label        = "return.<class>"   -- e.g. "return.large-cap"
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Rate }])
    -- Positive for return; negative for depreciation (e.g. vehicles)

RateStream (allocation weight for one asset class, e.g. "Large Cap Equities"):
    kind         = RateStream
    label        = "weight.<class>"   -- e.g. "weight.large-cap"
    owner        = Plan(plan_id)   -- or MemberAccount/JointAccount for account-level override
    start        = CalendarYear(plan.anchor_year)
    procedure    = Stored
    inputs       = {}
    value_schema = AttributeMap([{ key: "value", unit: Rate }])   -- fraction 0.0–1.0
```

Allocation weight streams may also be owned by `MemberAccount` or `JointAccount`
to override the plan-level weight for a specific account.

### Effective Rate Tree

The effective rate for an account is a small stream tree. The root is an
`Additive` RateStream owned by the account. Each asset class contributes a
weighted-return child:

```
RateStream (effective rate root for account A):
    kind         = RateStream
    owner        = Account(A.id)   -- or Plan for a shared plan-level effective rate
    start        = CalendarYear(<start_year>)
    procedure    = Additive
    inputs       = { "<asset-class-label>": weighted_return_stream_id, ... }
    -- one entry per asset class; labels are user-assigned (e.g. "large-cap", "bonds")

RateStream (WeightedReturn for one asset class):
    kind         = RateStream
    label        = "weighted-return.<class>"   -- e.g. "weighted-return.large-cap"
    owner        = Plan(plan_id)   -- or MemberAccount/JointAccount for account-level overrides
    procedure    = Product
    inputs       = { "weight": allocation_weight_stream_id, "rate": return_rate_stream_id }
    value_schema = AttributeMap([{ key: "value", unit: Rate }])
```

The account's balance `StreamTemplate` references the effective rate root via
`inputs["rate"]`. Adding a new asset class means creating a new weighted-return
stream and adding it as a named entry to the effective rate root's `inputs` — no
schema change.

**Allocation weight invariants:**
- The sum of all allocation weight stored points active for a given year must
  equal 1.0. This is a plan-construction requirement.
- An effective rate root with no weighted-return inputs is a plan-construction
  error. Zero is never a valid effective rate from an empty tree.

---

## Aggregate StreamTemplates

Aggregate `StreamTemplate`s define the rollup structure of the plan — which
balance templates roll up into which groupings. They use `procedure = Additive`
and their `inputs` map wires the children. The aggregate tree structure is plan
data, not engine logic. The `generate()` function creates the default aggregate
templates; users (and future plan-creation tools) may reorganize, add, or remove
aggregates by editing these templates. The engine does not classify accounts into
buckets — it reads the aggregate templates from the plan and evaluates them
uniformly.

There are no special aggregate types — an aggregate is simply an `Additive`
`StreamTemplate` whose `inputs` reference other streams. This extends to
user-defined aggregates: any `Additive` `StreamTemplate` wired to whatever child
streams the user chooses is a valid aggregate node.

The default MVP aggregate tree (created by `generate()`):

```
"net-worth" (Additive, owner = Plan)
    inputs = { "assets": assets_template, "liabilities": liabilities_template }

  "assets" (Additive, owner = Plan)
      inputs = { "retirement": retirement_template, "health": health_template,
                 "taxable": taxable_template, "hard-assets": hard_assets_template }

    "retirement" (Additive, owner = Plan)
        inputs = { "<account-label>": ..., ... }   -- 401k, Roth 401k, IRA, Roth IRA

    "health" (Additive, owner = Plan)
        inputs = { "<account-label>": ... }   -- HSA

    "taxable" (Additive, owner = Plan)
        inputs = { "<account-label>": ..., ... }   -- bank, brokerage

    "hard-assets" (Additive, owner = Plan)
        inputs = { "<entity-label>": ..., ... }   -- properties, vehicles

  "liabilities" (Additive, owner = Plan)
      inputs = { "lt-liabilities": lt_template, "st-liabilities": st_template }

    "lt-liabilities" (Additive, owner = Plan)
        inputs = { "<entity-label>": ..., ... }   -- AmortizedLoan, InterestOnlyLoan

    "st-liabilities" (Additive, owner = Plan)
        inputs = { "<entity-label>": ..., ... }   -- CreditLine
```

The leaf `inputs` in aggregate templates reference balance `StreamTemplate` IDs.
The labels (`"net-worth"`, `"retirement"`, etc.) identify aggregates for CLI
navigation.

---

## Projection Output Streams

The projection engine creates an ephemeral `Stream` for every `StreamTemplate`
in the plan — both balance templates and aggregate templates. The projection
tree mirrors the plan's template structure. All output stream points carry
denomination `YNV(point.year)`. Output streams are never persisted. They are
identified by `label` and navigated by `inputs` top-down.

Each projection leaf is a `Stream` created from the corresponding balance
`StreamTemplate` — it carries the template's `procedure`, `inputs`, and `owner`,
and produces `YNV(point.year)` output:

```
Stream (one per account, from account StreamTemplate):
    kind         = DollarStream
    label        = "balance"
    owner        = Account(account_id)
    procedure    = EndOfYearGrowth
    inputs       = { "rate": effective_rate_stream_id, "<label>": contribution_stream_id, ... }
    -- inputs from the plan's StreamTemplate for this account
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YNV(point.year)] end-of-year balance

Stream (one per property or vehicle, from entity StreamTemplate):
    kind         = DollarStream
    label        = "balance"
    owner        = Property(property_id) | Vehicle(vehicle_id)
    procedure    = EndOfYearGrowth
    inputs       = { "rate": rate_stream_id, "<label>": capital_improvement_stream_id, ... }
    -- inputs from the plan's StreamTemplate for this entity
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YNV(point.year)]

-- Projection leaf for a CreditLine:
Stream:
    kind         = DollarStream
    label        = "balance"
    owner        = Liability(credit_line_id)
    procedure    = EndOfYearGrowth
    inputs       = { "rate": interest_rate_stream_id, "<label>": payment_stream_id, ... }
    --   seed value is negative; non-"rate" inputs are payment streams (positive values reduce the balance)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YNV(point.year)]

-- Projection leaf for an AmortizedLoan:
Stream:
    kind         = DollarStream
    label        = "balance"
    owner        = Liability(amortized_loan_id)
    procedure    = AmortizedSchedule { cpi_stream_id, periods_per_year }
    inputs       = { "rate": interest_rate_stream_id, "payment": payment_stream_id, "<label>": prepayment_stream_id, ... }
    --   inputs from the plan's StreamTemplate for this loan
    --   seed value (P) is negative; recurrence derives remaining balance per year
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YNV(point.year)]

-- Projection leaf for an InterestOnlyLoan:
Stream:
    kind         = DollarStream
    label        = "balance"
    owner        = Liability(interest_only_loan_id)
    procedure    = InterestOnly
    inputs       = { "rate": interest_rate_stream_id }
    --   seed value is negative; balance constant, interest accrues separately
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YNV(point.year)]
```

Aggregate projection `Stream`s mirror the plan's aggregate `StreamTemplate`s.
The engine evaluates them all identically:
`emit(y) = sum(v.emit(y) for v in inputs.values())`.

**Balance recurrence formula** (end-of-year convention per
[D:formulas / balance-recurrence / end-of-year](../requirements/conceptual-model.md#d-formulas)):

The recurrence is defined only for `y > resolved_start`:

```
p(y) = p(y-1) × (1 + rate(y)) + net_flow(y)     [y > resolved_start]
```

where:
- `rate(y)` = `inputs["rate"].emit(y)` — the growth rate RateStream's value for year `y`
- `net_flow(y)` = sum of all non-`"rate"` input values for year `y`, each converted YZV→YNV; CPI resolution is built into the procedure

`effective_rate(y)` is computed per
[D:formulas / effective-rate](../requirements/conceptual-model.md#d-formulas):

```
effective_rate(y) = Σ( allocation_weight(c, y) × return_rate(c, y) )   for each asset class c
```

This is the `Additive` sum over `WeightedReturn` inputs, where each entry emits
`allocation_weight.emit(y) × return_rate.emit(y)` via `Product`.

**Base case** — the recurrence is undefined at `resolved_start`. The balance at
`resolved_start` is set from the seed `StreamPoint`. The seed's denomination
determines the conversion:

- `YZV` seed: `p(resolved_start) = seed × cpi_factors[resolved_start]`
- `PNV(ref_year)` seed: `p(resolved_start) = seed` (already nominal)

**Balance bound.** After applying the recurrence, the result is clamped toward
zero based on the stream's owner type:

```
-- asset streams (owner is Account, Property, or Vehicle): floor at zero
p(y) = max(p(y-1) × (1 + rate(y)) + net_flow(y), 0.0)

-- liability streams (owner is Liability): ceiling at zero
p(y) = min(p(y-1) × (1 + rate(y)) + net_flow(y), 0.0)
```

Without the ceiling, naively applying the recurrence past a liability's payoff
point would produce positive balances — implying the lender owes the borrower
money. The same ceiling applies to `AmortizedSchedule` streams.

**End-to-end numerical example:**

Inputs:
- `anchor_year = 2025`, `resolved_start = 2025`
- `seed = $10,000 [YZV]` (seed point in account StreamTemplate)
- One contribution DollarStream in inputs: `$1,000 [YZV]` per year (procedure=Stored)
- CPI RateStream (Stored): 3% in 2025, 4% in 2026
- Two WeightedReturn inputs on the effective rate root:
  - large-cap: AllocationWeight=0.60, ReturnRate=0.08
  - bonds: AllocationWeight=0.40, ReturnRate=0.04
- Effective rate root (Additive): sum of WeightedReturn inputs

Step 1 — derive `effective_rate(y)` (same for both years given constant inputs):

```
effective_rate(y) = 0.60 × 0.08 + 0.40 × 0.04
                  = 0.048 + 0.016
                  = 0.064   (6.4%)
```

Step 2 — base case. Since `resolved_start == anchor_year`, the CPI factor is 1.0:

```
p(2025) = $10,000   [YNV(2025)]
```

Step 3 — project year 2026. Convert the contribution input from YZV to YNV(2026):

```
net_flow_ynv(2026) = $1,000 × 1.03 = $1,030   [YNV(2026)]
```

Apply the recurrence:

```
p(2026) = $10,000 × 1.064 + $1,030 = $11,670   [YNV(2026)]
```

Step 4 — project year 2027:

```
net_flow_ynv(2027) = $1,000 × 1.03 × 1.04 = $1,071.20   [YNV(2027)]
p(2027) = $11,670 × 1.064 + $1,071.20 = $13,488.08   [YNV(2027)]
```

---

## Design Invariants

1. All asset and assumption entity dollar fields are denominated in
   `anchor_year` dollars (`[YZV]`). Liability entity dollar fields (principal,
   balance, payment, credit limit) are denominated in nominal dollars at the
   entity's `start_year` (see [Liabilities § Denomination convention](#liabilities)).
   No `CNV` values may be stored in input entity fields.

2. `CNV` never appears on any stored `StreamPoint`. CNV is a display-layer denomination. CNV conversion is FUT; the MVP engine outputs YNV only.

3. For all projection output `DollarStream` points, `denomination = YNV(point.year)`.
   The denomination of any stored point is determinable from the point record
   alone.

4. The sum of all `AllocationWeight` RateStream values active for a given year
   and account must equal 1.0. This is evaluated across all `WeightedReturn`
   inputs of the effective rate root for that account and year. A zero-sum
   result (no active WeightedReturn inputs, or all weights zero) is a
   plan-construction error. Zero is never a valid in-scope effective rate from
   an empty or zero-weight allocation tree.

5. Every `MemberAge` start references a `member_id` that exists in the plan's
   household. Dangling member references are invalid.

6. `Member.birth_year` is immutable after plan creation.

7. No age-derived calendar year (i.e., `birth_year + age`) may be stored as a
   raw `start_year` or `end_year` on a stream record. Age-relative starts are
   expressed as `StreamStart::MemberAge`. Age-relative ends are expressed via
   `TerminationRef::OnEvent`.

8. No stream record stores an explicit end year or end age. Stream ends are
   derived from the ownership chain per the Stream Lifecycle rules.

9. `MemberLifecycleStream` streams are resolved before all streams that reference
   them in each projection year. This is the canonical engine resolution order.

10. A spousal relationship (`RelationSpouseStream`) requires exactly one other member
    with `MemberRole = Primary` or `MemberRole = Spouse` in the household. At
    most one spousal pair is permitted per plan.

11. A `StreamTemplate`'s `start` year must be `>=` the year of its seed
    `StreamPoint`. The projection starts at `resolved_start`; the seed must
    exist at or before that year for the base case to resolve.

12. Every `AttributeDef` with a carry-forward unit (`Rate`, `Count`, `Years`,
    or `Age`) must have a non-`None` `initial` value. This is a plan-construction
    requirement enforced at load time. The `initial` value is the terminal
    fallback for carry-forward resolution and is not tied to any year.

13. The pair `(stream_id, year)` is unique across all `StreamPoint` records. A
    stream may have at most one point per year.

14. The pair `(owner, label)` is unique across all `Stream` records within a
    plan. Two streams with the same owner may not share a label.

15. Entity labels (`Account.label`, `Property.label`, `Vehicle.label`, and
    liability entity labels) must be globally unique within a plan.

---

## UUID Derivation

All entity and stream UUIDs are UUID v5 (SHA-1 namespace). A fixed YARP
namespace UUID is baked into the binary. The name input is derived from the
entity's identity:

- **Plan** — `"plan:{name}"`
- **Household** — `"{plan_id}:household"`
- **Member** — `"{household_id}:member:{given_name}.{family_name}"`
- **LifecycleEvent** — `"{member_id}:event:{kind}"` (e.g., `"{member_id}:event:retirement"`)
- **Labeled entities** (Account, Property, Vehicle, liabilities) —
  `"{parent_id}:{entity_type}:{label}"` (e.g., `"{plan_id}:account:Alice 401k"`)
- **Streams** — `"{owner_id}:{label}"` (e.g., `"{account_id}:balance"`)
  For composite owners (`MemberAccount`, `Relationship`), `owner_id` is the
  concatenation of the constituent UUIDs (e.g., `"{member_id}:{account_id}"`).
- **StreamPoints** — `"{stream_id}:{year}"`

Because entity labels are globally unique (invariant 19) and `(owner, label)`
is unique for streams (invariant 18), this scheme produces deterministic,
collision-free UUIDs that are stable across projection tree reorganization.

---

## Deferred: Retirement Budget

Income sources (Social Security benefits, pensions, wages) and expenses
(living, health, housing, discretionary) are not modeled as streams. They
are configuration inputs to the retirement budget computation. Entity schemas
for income and expense parameters, and how they feed into retirement income
planning, will be specified when the retirement budget concept is designed.

---

## Open Design Questions

These questions do not block MVP but require decisions before the affected
features are implemented.

- **YNV backward projection from YZV:** Bedrock principles note this is an open
  design question. The data model does not assume backward projection is supported.
  If added, projection output streams will need a backward-sweep variant and the
  denomination rules will need to address deflation below the anchor year.

---

## Cross-Reference Index

This table maps `S:` paths from the conceptual model to their design records
in this document. Only paths with a corresponding MVP or CORE `R:` requirement
are included; FUT and unanchored paths are omitted.

| Conceptual Model Path | Anchor Record | StreamKind |
|-----------------------|---------------|------------|
| `assumptions / inflation / cpi` | — | `RateStream` (label=`"cpi"`, Stored, plan-level) |
| `assumptions / rates / large-cap` | — | `RateStream` (label=`"return.large-cap"`) |
| `assumptions / rates / small-cap` | — | `RateStream` (label=`"return.small-cap"`) |
| `assumptions / rates / international` | — | `RateStream` (label=`"return.international"`) |
| `assumptions / rates / bonds` | — | `RateStream` (label=`"return.bonds"`) |
| `assumptions / rates / real-estate` | — | `RateStream` (label=`"return.real-estate"`) |
| `assumptions / rates / vehicle` | — | `RateStream` (label=`"return.vehicle"`) |
| `assumptions / allocations` | — | `RateStream` (label=`"weight.<class>"`, owner=Plan; account overrides use same kind) |
| `assumptions / allocations / [ member ] / [ account ]` | — | `RateStream` (label=`"weight.<class>"`, owner=MemberAccount) |
| `assumptions / allocations / [ account ]` | — | `RateStream` (label=`"weight.<class>"`, owner=JointAccount) |
| `household / [ member ]` | `Member` | `MemberLifecycleStream` |
| `household / [ member ] / relations / spouse` | — | `RelationSpouseStream` |
| `assets / accounts / [ member ] / [ 401k ]` | `Account` (kind=Traditional401k) | `StreamTemplate` (label=`"balance"`) |
| `assets / accounts / [ member ] / [ roth-401k ]` | `Account` (kind=Roth401k) | `StreamTemplate` (label=`"balance"`) |
| `assets / accounts / [ member ] / [ ira ]` | `Account` (kind=TraditionalIra) | `StreamTemplate` (label=`"balance"`) |
| `assets / accounts / [ member ] / [ roth-ira ]` | `Account` (kind=RothIra) | `StreamTemplate` (label=`"balance"`) |
| `assets / accounts / [ member ] / [ hsa ]` | `Account` (kind=Hsa) | `StreamTemplate` (label=`"balance"`) |
| `assets / accounts / [ member ] / [ bank ]` | `Account` (kind=Bank, owner=MemberOwned) | `StreamTemplate` (label=`"balance"`) |
| `assets / accounts / [ member ] / [ brokerage ]` | `Account` (kind=Brokerage, owner=MemberOwned) | `StreamTemplate` (label=`"balance"`) |
| `assets / accounts / [ bank ]` | `Account` (kind=JointBank, owner=JointOwned) | `StreamTemplate` (label=`"balance"`) |
| `assets / accounts / [ brokerage ]` | `Account` (kind=JointBrokerage, owner=JointOwned) | `StreamTemplate` (label=`"balance"`) |
| `assets / hard-assets / properties / [ home ]` | `Property` (kind=PrimaryResidence) | `StreamTemplate` (label=`"balance"`) |
| `assets / hard-assets / properties / [ vacation-home ]` | `Property` (kind=VacationHome) | `StreamTemplate` (label=`"balance"`) |
| `assets / hard-assets / properties / [ parcel ]` | `Property` (kind=LandParcel) | `StreamTemplate` (label=`"balance"`) |
| `assets / hard-assets / [ vehicle ]` | `Vehicle` | `StreamTemplate` (label=`"balance"`) |
| `contributions / [ member ] / [ 401k ] / employee` | — | `DollarStream` (Stored, inputs key `"employee-401k"`) |
| `contributions / [ member ] / [ 401k ] / employer` | — | `DollarStream` (Stored, inputs key `"employer-401k"`) |
| `contributions / [ member ] / [ 401k ] / catchup` | — | `DollarStream` (Stored, inputs key `"catchup-401k"`) |
| `contributions / [ member ] / [ roth-401k ] / employee` | — | `DollarStream` (Stored, inputs key `"employee-roth-401k"`) |
| `contributions / [ member ] / [ roth-401k ] / employer` | — | `DollarStream` (Stored, inputs key `"employer-roth-401k"`) |
| `contributions / [ member ] / [ roth-401k ] / catchup` | — | `DollarStream` (Stored, inputs key `"catchup-roth-401k"`) |
| `contributions / [ member ] / [ ira ] / standard` | — | `DollarStream` (Stored, inputs key `"standard-ira"`) |
| `contributions / [ member ] / [ ira ] / catchup` | — | `DollarStream` (Stored, inputs key `"catchup-ira"`) |
| `contributions / [ member ] / [ roth-ira ] / standard` | — | `DollarStream` (Stored, inputs key `"standard-roth-ira"`) |
| `contributions / [ member ] / [ roth-ira ] / catchup` | — | `DollarStream` (Stored, inputs key `"catchup-roth-ira"`) |
| `contributions / [ member ] / [ hsa ] / employee` | — | `DollarStream` (Stored, inputs key `"employee-hsa"`) |
| `contributions / [ member ] / [ hsa ] / employer` | — | `DollarStream` (Stored, inputs key `"employer-hsa"`) |
| `contributions / [ member ] / [ hsa ] / catchup` | — | `DollarStream` (Stored, inputs key `"catchup-hsa"`) |
| `contributions / [ member ] / [ bank ]` | — | `DollarStream` (Stored, inputs key `"bank"`) |
| `contributions / [ member ] / [ brokerage ]` | — | `DollarStream` (Stored, inputs key `"brokerage"`) |
| `contributions / [ bank ]` | — | `DollarStream` (Stored, inputs key `"bank"`, owner=JointAccount) |
| `contributions / [ brokerage ]` | — | `DollarStream` (Stored, inputs key `"brokerage"`, owner=JointAccount) |
| `liabilities / [ amortized-loan ]` | `AmortizedLoan` | `StreamTemplate` (label=`"balance"`) |
| `liabilities / [ interest-only-loan ]` | `InterestOnlyLoan` | `StreamTemplate` (label=`"balance"`) |
| `liabilities / [ credit-line ]` | `CreditLine` | `StreamTemplate` (label=`"balance"`) |
