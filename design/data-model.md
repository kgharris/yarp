# Data Model

This document defines the data model for the YARP retirement planning engine. It
is anchored to the [conceptual model](../requirements/conceptual-model.md) and
the [bedrock principles](../bedrock-principles.md). Every design decision here
traces to an `S:`, `D:`, or `R:` path in those documents.

The stream schema is defined first. All other schemas are derived from it.

---

## Stream Primitives

Every time-varying quantity in the system is a stream. This schema applies
uniformly to all streams regardless of what domain concept they represent.

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
4. Else (plan-owned and policy streams) → extends to the plan timeline end,
   which is itself derived from all member lifetimes.

**Why no explicit ends on streams:**

Storing an explicit end year or end age on a stream creates a redundant fact
that can drift out of sync with the entity data it is derived from. If the 401k
contribution stream stored `end_age: M.retirement_age`, it would require
updating whenever `Member.retirement_age` changes. With event-driven
termination, there is one authoritative fact (the `LifecycleEvent.age`), one
owner (the event entity), and every stream that references the event terminates
at the same derived year automatically.

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
member.retirement_age`. Its `age` field holds the retirement age directly —
changing `retirement_age` on the member means creating or updating the event,
which then cascades to all streams that reference it.

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
    owner:        OwnedBy,
    start:        StreamStart,             -- when the stream becomes active
    terminates:        Option<TerminationRef>,       -- None = ends at natural bound (see Stream Lifecycle)
    parent_id:    Option<Uuid>,
    value_schema: ValueSchema,
}
```

**Field notes:**

- `kind` — classifies what domain concept this stream represents (see
  [Stream Kind Registry](#stream-kind-registry) below). Drives polymorphic
  behavior in the engine without embedding that behavior in the schema.
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
- `parent_id` — composition link. A stream with a `parent_id` is a child stream
  contributing to its parent. The parent aggregates children according to its
  `StreamKind`'s composition rule.
- `value_schema` — the named attributes this stream's points carry. Every stream
  uses `AttributeMap`; streams with a single time-varying quantity use a single-entry
  map with key `"value"`.

```
ValueSchema = AttributeMap(attributes: [AttributeDef])

AttributeDef {
    key:  string,
    unit: ValueUnit,
}

ValueUnit = Decimal | Rate | Years | Count | Age
```

The denomination (`YZV`, `PNV`, or `YNV`) is carried on the `StreamPoint`, not
on the `ValueSchema`.

### Identity Semantics

Every stream emits a value for every year in the plan timeline. For years with
no explicit anchor point — whether before the stream's start, after its derived
end, or within gaps — the stream emits an **identity value**. The identity is
determined by the attribute's `ValueUnit`:

| Unit | Identity | Rationale |
|------|----------|-----------|
| `Decimal` | `0` | Additive identity — contributes nothing to any sum |
| `Rate` | carry-forward | Most recent explicit anchor; recurse backward until one is found |
| `Count` | carry-forward | Enumeration values hold until a new anchor supersedes them |
| `Years` | carry-forward | Same as `Rate` |
| `Age` | carry-forward | Same as `Rate` |

Carry-forward always terminates at the stream's first explicit anchor point.
A stream with no explicit anchor points at all is a data error caught at plan
construction, not at projection time.

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

### Composition Rules

Child streams compose into their parent. The aggregation semantics depend on the
parent stream's `StreamKind`:

```
CompositionRule =
    | Additive      -- parent value = sum of active child values
    | Delegated     -- parent is a container; engine reads children directly
    | Override      -- child points override parent; carry-forward semantics apply
```

| Parent Kind | Composition Rule |
|-------------|-----------------|
| `AccountBalanceStream` | `Additive` — net cash flow from child `CashFlow` streams applied to the balance recurrence formula |
| `ExpenseLiving`, `ExpenseHousing`, etc. | `Additive` — sum of child expense streams |
| `AllocationStream` | `Delegated` — if an account has an `AllocationStream` defined (owner = Account or MemberAccount), the engine reads allocations from it. Otherwise it reads from the plan-level `AllocationStream` (owner = Plan). The choice is structural (stream exists or not), not value-based. |
| Policy streams | `Override` — child streams are per-year override points; carry-forward semantics resolve the effective value |
| Projection output streams | Not composited; the engine writes directly |

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
Plan-level and policy streams extend to `timeline_end`.

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

`StreamKind` classifies every stream into a domain category. This drives engine
dispatch without embedding domain logic in the schema.

The full taxonomy:

```
StreamKind =
    -- Household
    | MemberLifecycleStream
    | RelationSpouseStream

    -- Accounts
    | AccountBalanceStream

    -- Cash flows (contributions and withdrawals)
    | ContributionStream
    | Withdrawal401kStandardStream
    | WithdrawalRoth401kStream
    | WithdrawalIraStandardStream
    | WithdrawalRothIraStream
    | WithdrawalHsaQualifiedStream
    | WithdrawalBankStream
    | WithdrawalBrokerageStream

    -- Liabilities
    | LoanBalanceStream
    | LoanPaymentStream
    | InterestOnlyBalanceStream
    | InterestOnlyPaymentStream
    | CreditLineBalanceStream
    | CreditLinePaymentStream

    -- Hard assets
    | PropertyValueStream
    | VehicleValueStream

    -- Assumptions — inflation
    | InflationCpiStream

    -- Assumptions — return rates
    | ReturnRateLargeCapStream
    | ReturnRateSmallCapStream
    | ReturnRateInternationalStream
    | ReturnRateBondsStream
    | ReturnRateRealEstateStream
    | DepreciationRateVehicleStream

    -- Assumptions — allocations
    | AllocationStream

    -- Policy — HSA
    | PolicyHsaLimitsStream

    -- Projection outputs (ephemeral; computed on demand, never persisted)
    | ProjectionAccountBalanceStream
```

**Note on `AccountType`:** The `AccountBalanceStream` kind is common to all
account types. The account type is carried on the `Account` entity, not on the
stream kind. The engine dispatches on `Account.account_kind`, not on a
stream-kind variant per account type.

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

AssetClass = LargeCap | SmallCap | International | Bonds | RealEstate

FilingStatus = Single | MarriedFilingJointly

MemberRole = Primary | Spouse

PropertyKind = PrimaryResidence | VacationHome | LandParcel

LiabilityKind = AmortizedLoan | InterestOnlyLoan | CreditLine

HsaCoverageType = SelfOnly | Family

ContributionRole =
    -- 401k
      Employee401k | Employer401k | Catchup401k
    -- Roth 401k
    | EmployeeRoth401k | EmployerRoth401k | CatchupRoth401k
    -- IRA / Roth IRA
    | StandardIra | CatchupIra | StandardRothIra | CatchupRothIra
    -- HSA
    | EmployeeHsa | EmployerHsa | CatchupHsa
    -- Bank / Brokerage
    | Bank | Brokerage
```

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
    retirement_age: Option<i32>,     -- None if not applicable
}
```

`birth_year` is the member's single calendar anchor. Every derived calendar
event — retirement — is computed as `birth_year + age`. Storing
ages rather than calendar years means that correcting `birth_year` automatically
shifts all downstream events.

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
        { key: "age",   unit: Age   },
        { key: "phase", unit: Count },  -- 0=working, 1=retired (dimensionless enum)
    ])
```

The `phase` attribute transitions from `working` to `retired` at
`birth_year + retirement_age`. Consumers read the attribute from the stream;
no branching on age in consumer code.

**Plan-construction invariant:** `MemberLifecycleStream` must have an explicit
anchor point at `age = 0` with `phase = 0` (working). This anchor is required;
without it the carry-forward rule has no initial value and pre-retirement phase
is undefined.

**Design invariant — resolution order:** `MemberLifecycleStream` streams are resolved
before all streams that reference them (cash inflow streams, contribution bounds,
health insurance phase transitions). This is a named engine contract: the
engine's projection pass resolves household lifecycle streams first in each year
before evaluating any stream that depends on a member's phase or age attribute.

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
    opening_year:    i32,
    closing_year:    Option<i32>,
    coverage_type:   Option<HsaCoverageType>,  -- required when account_kind = Hsa; None for all other account kinds
}
```

### Account Balance Stream

Each account has exactly one `AccountBalanceStream` stream. This stream is the root
of all cash flow child streams for the account.

```
AccountBalanceStream for account A:
    kind         = AccountBalanceStream
    owner        = Account(A.id)
    start        = CalendarYear(A.opening_year)
    -- end derived from Account.closing_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- Stored input points (user-entered seed balances) are YZV.
    -- Projection output points for this account are separate:
    --   see ProjectionAccountBalanceStream in Projection Output Streams below.
```

**Denomination separation:** The `AccountBalanceStream` stream holds user-entered
seed values in `YZV`. The engine seeds projection from the `AccountBalanceStream`
YZV value. Projected year-end balances are written by the engine to a separate
`ProjectionAccountBalanceStream` stream (denomination `YNV(year)`). A single stream
does not carry two denominations.

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
}
```

```
PropertyValueStream for property P:
    kind         = PropertyValueStream
    owner        = Property(P.id)
    start        = CalendarYear(P.purchase_year)
    -- end derived from Property.sale_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] seed; projected YNV per year by engine
    -- growth rate sourced from ReturnRateRealEstateStream assumption stream
```

### Vehicle

```
Vehicle {
    id:               Uuid,
    plan_id:          Uuid,
    label:            string,
    purchase_year:    i32,
    disposal_year:    Option<i32>,
    value_yzv:        Decimal,      -- [YZV] estimated value at purchase_year
}
```

```
VehicleValueStream for vehicle V:
    kind         = VehicleValueStream
    owner        = Vehicle(V.id)
    start        = CalendarYear(V.purchase_year)
    -- end derived from Vehicle.disposal_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] seed; projected YNV per year
    -- depreciation rate sourced from DepreciationRateVehicleStream assumption stream
```

---

## Contributions

Contribution streams are owned by accounts. Each contribution type is a
distinct stream kind. Accounts relate to contribution streams via a
one-to-many relationship keyed on `(account_id, kind)`. The engine resolves
contributions for an account by querying all contribution streams owned by
that account.

All contribution streams share a single `kind = ContributionStream`. The `role`
attribute identifies the contribution type for future limit enforcement; the engine
dispatches to the appropriate limit bucket by reading `role`. All values are
`[YZV]`; positive = inflow.

### 401k Contributions

```
ContributionStream for member M on account A (Employee401k):
    kind         = ContributionStream
    role         = Employee401k
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(retirement_event_id)
    -- retirement_event_id references LifecycleEvent { kind: Retirement, age: M.retirement_age }
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

ContributionStream for member M on account A (Employer401k):
    kind         = ContributionStream
    role         = Employer401k
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(retirement_event_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

ContributionStream for member M on account A (Catchup401k):
    kind         = ContributionStream
    role         = Catchup401k
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: 50)
    terminates   = OnEvent(retirement_event_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- When limits are in scope: age-tiered limit applies (ages 50–59 and 64+: standard
    --   catchup; ages 60–63: SECURE 2.0 super-catchup). Role is sufficient to dispatch.
```

### Roth 401k Contributions

```
ContributionStream for member M on account A (EmployeeRoth401k):
    kind         = ContributionStream
    role         = EmployeeRoth401k
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(retirement_event_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

ContributionStream for member M on account A (EmployerRoth401k):
    kind         = ContributionStream
    role         = EmployerRoth401k
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(retirement_event_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

ContributionStream for member M on account A (CatchupRoth401k):
    kind         = ContributionStream
    role         = CatchupRoth401k
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: 50)
    terminates   = OnEvent(retirement_event_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- Combined catchup limit shared with Catchup401k per member when limits are in scope.
```

### IRA Contributions

```
ContributionStream for member M on account A (StandardIra):
    kind         = ContributionStream
    role         = StandardIra
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: 0)
    -- no terminates — ends at member death (natural bound)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

ContributionStream for member M on account A (CatchupIra):
    kind         = ContributionStream
    role         = CatchupIra
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: 50)
    -- no terminates — ends at member death (natural bound)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
```

### Roth IRA Contributions

```
ContributionStream for member M on account A (StandardRothIra):
    kind         = ContributionStream
    role         = StandardRothIra
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: 0)
    -- no terminates — ends at member death (natural bound)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

ContributionStream for member M on account A (CatchupRothIra):
    kind         = ContributionStream
    role         = CatchupRothIra
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: 50)
    -- no terminates — ends at member death (natural bound)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- Combined IRA catchup limit shared with CatchupIra per member when limits are in scope.
```

### HSA Contributions

```
ContributionStream for member M on account A (EmployeeHsa):
    kind         = ContributionStream
    role         = EmployeeHsa
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(medicare_event_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

ContributionStream for member M on account A (EmployerHsa):
    kind         = ContributionStream
    role         = EmployerHsa
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: <employment_start_age>)
    terminates   = OnEvent(medicare_event_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])

ContributionStream for member M on account A (CatchupHsa):
    kind         = ContributionStream
    role         = CatchupHsa
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    start        = MemberAge(M.id, age: 55)
    -- no terminates — ends when HDHP coverage ends (modeled via natural bound for MVP)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
```

### Bank and Brokerage Contributions

```
ContributionStream (Bank or Brokerage):
    kind         = ContributionStream
    role         = Bank | Brokerage
    owner        = MemberAccount(M.id, A.id) | JointAccount(A.id)
    parent_id    = A's AccountBalanceStream id
    -- no start, no terminates — open-ended; owner discriminates member vs joint
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
```

---

## Withdrawals

Withdrawal streams are owned by accounts, parallel to contributions. The engine
resolves withdrawals for an account by querying all withdrawal streams for that
account.

**Sign convention:** withdrawal stream values are stored as **negative decimals**.
This is a storage convention — the user enters a withdrawal amount and the system
stores it as a negative value. Contributions are stored as positive values.
`net_flow(y)` is therefore their algebraic sum: no subtraction step is needed in
the formula.

### 401k Withdrawals

```
Withdrawal401kStandardStream:
    kind         = Withdrawal401kStandardStream
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    -- voluntary; user-configurable amount per year
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; negative = outflow
```

### Roth 401k Withdrawals

```
WithdrawalRoth401kStream:
    kind         = WithdrawalRoth401kStream
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    -- qualified withdrawals; tax-free
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; negative = outflow
```

### IRA Withdrawals

```
WithdrawalIraStandardStream:
    kind         = WithdrawalIraStandardStream
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    -- voluntary; user-configurable amount per year
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; negative = outflow
```

### Roth IRA Withdrawals

```
WithdrawalRothIraStream:
    kind         = WithdrawalRothIraStream
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    -- qualified withdrawals; tax-free
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; negative = outflow
```

### HSA Withdrawals

```
WithdrawalHsaQualifiedStream:
    kind         = WithdrawalHsaQualifiedStream
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalanceStream id
    -- end derived from account ownership chain (natural account end)
    -- matched against eligible health expenses; tax-free
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; negative = outflow
```

### Bank and Brokerage Withdrawals

```
WithdrawalBankStream and WithdrawalBrokerageStream:
    -- owner discriminates MemberAccount from JointAccount
    -- denomination = YZV; negative = outflow
```

---

## Liabilities

### Amortized Loan

```
AmortizedLoan {
    id:             Uuid,
    plan_id:        Uuid,
    label:          string,
    start_year:     i32,
    end_year:       i32,             -- payoff year
    principal:      Decimal,         -- [YZV] opening principal
    interest_rate:  Decimal,         -- dimensionless; annual rate
    payment_amount: Decimal,         -- [YZV] annual payment amount
}
```

```
LoanBalanceStream:
    kind         = LoanBalanceStream
    owner        = Liability(AmortizedLoan.id)
    start        = CalendarYear(start_year)
    -- end derived from AmortizedLoan.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] remaining principal per year

LoanPaymentStream:
    kind         = LoanPaymentStream
    owner        = Liability(AmortizedLoan.id)
    start        = CalendarYear(start_year)
    -- end derived from AmortizedLoan.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] annual payment (principal + interest)
```

### Interest-Only Loan

```
InterestOnlyLoan {
    id:            Uuid,
    plan_id:       Uuid,
    label:         string,
    start_year:    i32,
    end_year:      Option<i32>,
    balance:       Decimal,    -- [YZV] outstanding balance
    interest_rate: Decimal,    -- dimensionless; annual rate
}
```

```
InterestOnlyBalanceStream:
    kind         = InterestOnlyBalanceStream
    owner        = Liability(InterestOnlyLoan.id)
    start        = CalendarYear(start_year)
    -- end derived from InterestOnlyLoan.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]

InterestOnlyPaymentStream:
    kind         = InterestOnlyPaymentStream
    owner        = Liability(InterestOnlyLoan.id)
    start        = CalendarYear(start_year)
    -- end derived from InterestOnlyLoan.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] annual interest payment
```

### Credit Line

```
CreditLine {
    id:            Uuid,
    plan_id:       Uuid,
    label:         string,
    start_year:    i32,
    end_year:      Option<i32>,
    credit_limit:  Decimal,    -- [YZV]
    interest_rate: Decimal,    -- dimensionless
}
```

```
CreditLineBalanceStream:
    kind         = CreditLineBalanceStream
    owner        = Liability(CreditLine.id)
    start        = CalendarYear(start_year)
    -- end derived from CreditLine.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] revolving balance per year

CreditLinePaymentStream:
    kind         = CreditLinePaymentStream
    owner        = Liability(CreditLine.id)
    start        = CalendarYear(start_year)
    -- end derived from CreditLine.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] annual minimum or configured payment
```

---

## Assumption Streams

User-configurable assumption streams are stored as streams owned by the plan.
They use carry-forward semantics: the most recent point at or before a given
year is the effective value for that year. Per-year overrides are additional
points.

### Inflation Assumptions

One inflation stream per plan:

```
InflationCpiStream:
    kind         = InflationCpiStream
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([{ key: "value", unit: Rate }])   -- dimensionless; e.g., 0.03
```

A single CPI rate is applied uniformly to all streams that inflate over time.

### Return Rate Assumptions

One stream per asset class; all have the same structure:

```
ReturnRateLargeCapStream:
    kind         = ReturnRateLargeCapStream
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([{ key: "value", unit: Rate }])   -- dimensionless; annual expected nominal return (not real)

-- ReturnRateSmallCapStream, ReturnRateInternationalStream, ReturnRateBondsStream,
-- ReturnRateRealEstateStream follow the same structure.

DepreciationRateVehicleStream:
    kind         = DepreciationRateVehicleStream
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([{ key: "value", unit: Rate }])   -- dimensionless; negative rate (e.g., -0.15 = 15%/year depreciation)
```

### Allocation Assumptions

Allocation streams are multi-attribute. Each point carries a named fraction per
asset class; fractions are dimensionless (0.0–1.0) and must sum to 1.0 for every
explicitly stored point.

Allocations are **not** meaningful in isolation — they are a lookup table used
during dollar-stream projection. For each year the engine projects an account
balance, it looks up the effective allocation for that year and derives a weighted
return rate. The allocation itself never appears in output.

**Lookup semantics within an active stream:**
Allocation attributes use `unit: Rate`, giving them carry-forward identity
semantics. If a year within the stream's active range has no stored point, the
engine carries forward the most recently stored point (recursing backward until
one is found). This means a user who sets allocations once at age 30 gets those
same allocations in every subsequent year unless they add another anchor. Every
`AllocationStream` must have at least one stored point — a stream with no anchor
points at all is a plan-construction error.

**Lookup outside an active stream:**
If the engine looks up an allocation for a year outside the stream's active
range (before `start` or after its derived end), the stream is not applicable
for that year. The engine treats this the same as no stream existing for that
account-year: it falls back to the plan-level `AllocationStream`. For years
before the plan-level stream's `start`, there is no active account to project —
no lookup occurs.

**Zero is never a valid in-scope allocation.** An all-zeros result from an
allocation lookup is a data error — it means either the stream has no anchor
points (plan-construction error) or a bug in stream resolution. The engine must
never use a zero-sum allocation to compute an effective rate.

```
AllocationStream:
    kind         = AllocationStream
    value_schema = AttributeMap([
        { key: "large_cap",     unit: Rate },   -- fraction 0.0–1.0; carry-forward
        { key: "small_cap",     unit: Rate },
        { key: "international", unit: Rate },
        { key: "bonds",         unit: Rate },
        { key: "real_estate",   unit: Rate },
    ])
    -- Invariant: sum of all attribute values = 1.0 for each stored point
    -- Invariant: at least one stored point must exist (plan-construction requirement)

-- Plan-level global (one per plan):
    owner  = Plan(plan_id)
    start  = CalendarYear(plan.anchor_year)
    -- end derived from plan timeline (rule 4)

-- Account-level override (optional; one per account):
    owner  = MemberAccount(member_id, account_id)  -- for member-owned accounts
          | JointAccount(account_id)                -- for joint accounts
    start  = MemberAge(member_id, age)  -- for member-owned; encodes age-relative overrides
          | CalendarYear(year)           -- for joint accounts
    -- end derived from owner's natural bound per stream lifecycle rules
```

The engine resolves the effective allocation for an account based on stream
existence: if an `AllocationStream` is defined for the account, the engine reads
allocations from it for the years that stream is active. Otherwise it reads from
the plan-level `AllocationStream`. An account either has its own allocation stream
or it doesn't — there is no per-year switching between the two.

---

## Policy Data

Policy streams carry externally-sourced regulatory values. They are stored as
streams so that shipped defaults populate current and near-future years, and
per-year overrides let users model legislative changes without code changes.
The engine consumes them identically to user assumption streams.

All policy streams are owned by the plan. Historical values for past years are
`PNV(ref_year)`. Future-year defaults shipped with the application are `YZV`,
inflated to each target year (`YNV`) at runtime.

### Policy Streams — HSA

```
PolicyHsaLimitsStream:
    kind         = PolicyHsaLimitsStream
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([
        { key: "individual", unit: Decimal },  -- [YZV] annual limit for self-only HDHP coverage
        { key: "family",     unit: Decimal },  -- [YZV] annual limit for family HDHP coverage
        { key: "catchup",    unit: Decimal },  -- [YZV] additional limit for members age 55+; fixed by statute, never inflation-indexed
    ])
    -- The engine applies the limit matching Account.coverage_type for each member's HSA.
    -- The catchup attribute is a fixed amount; the engine ignores any inflation applied to it.
```

---

## Projection Output Streams

The projection engine writes its results as streams. Output streams are ephemeral —
computed on demand and never persisted. All output stream points carry denomination
`YNV(point.year)`.

```
ProjectionAccountBalanceStream (per account):
    kind         = ProjectionAccountBalanceStream
    owner        = Account(account_id)
    start        = CalendarYear(account.opening_year)
    -- end derived from plan timeline end (rule 4)
    value_schema = AttributeMap([
        { key: "value",         unit: Decimal },   -- [YNV(point.year)] end-of-year balance
        { key: "effective_rate", unit: Rate },      -- dimensionless; weighted nominal return for this account and year
    ])
    -- Each point: denomination = YNV(point.year)
    -- Distinct from AccountBalanceStream; that stream holds YZV seed inputs.
```

**Balance recurrence formula** (end-of-year convention per
[D:formulas / balance-recurrence / end-of-year](../requirements/conceptual-model.md#d-formulas)):

The recurrence is defined only for `y > opening_year`:

```
p(y) = p(y-1) * (1 + effective_rate(y)) + net_flow(y)     [y > opening_year]
```

where `effective_rate(y)` is the weighted average of return rates using the
resolved allocation stream for that account and year, and `net_flow(y)` is the
algebraic sum of all contribution (positive) and withdrawal (negative) stream
values for that year, converted from `YZV` to `YNV(y)` before the recurrence
is applied.

**Base case — the recurrence is undefined at `opening_year`.** The balance at
`opening_year` is not computed by the recurrence; it is set directly from the
user-entered opening balance stored in `AccountBalanceStream`, converted from
`YZV` to `YNV(opening_year)`:

```
p(opening_year) = seed_yzv * ∏(1 + r_t)   for t = anchor_year, …, opening_year−1
```

where `seed_yzv` is the value stored in `AccountBalanceStream` for this account.
If `opening_year == anchor_year`, the product is empty and `p(opening_year) = seed_yzv`
(no inflation has elapsed). The recurrence then proceeds from `y = opening_year + 1`
onward using `p(opening_year)` as `p(y-1)`.

The YZV-to-YNV conversion for a stream value in year `y` is the per-year product
of CPI rates from `InflationCpiStream`:

```
value_ynv = value_yzv * ∏(1 + r_t)   for t = anchor_year, anchor_year+1, …, y−1
```

where `r_t` is the rate emitted by `InflationCpiStream` for year `t`. Each year
contributes its own rate; a single scalar exponent is only correct when CPI is
constant across the entire projection window, which cannot be assumed.

Example: `anchor_year = 2025`, `value_yzv = $10,000`, CPI rates: 3% in 2025,
4% in 2026.
- Converting to YNV(2026): `$10,000 * (1.03) = $10,300`
- Converting to YNV(2027): `$10,000 * (1.03) * (1.04) = $10,712`

**Balance floor.** After applying the recurrence, the result is clamped at zero:

```
p(y) = max(p(y-1) * (1 + effective_rate(y)) + net_flow(y), 0.0)
```

The value stored in `ProjectionAccountBalanceStream` for year `y` is never
negative. In MVP (projection-only, no withdrawal sequencer), the floor can only
be reached if net cash flow drives the balance below zero; no shortfall output
is defined. When withdrawals are in scope, sequencing must ensure withdrawals
do not exceed the available balance before the recurrence is applied.

---

## Design Invariants

1. All `[YZV]`-denominated entity fields contain values denominated in
   `anchor_year` dollars. No `CNV` or `YNV` values may be stored in input
   entity fields.

2. `CNV` never appears on any stored `StreamPoint`. CNV is a display-layer denomination. CNV conversion is FUT; the MVP engine outputs YNV only.

3. For projection output stream points, `denomination = YNV(point.year)`.
   The denomination of any stored point is determinable from the point record
   alone.

4. All `AllocationStream` stored points must have attribute values summing to 1.0.
   Every `AllocationStream` must have at least one stored point — a stream with no
   anchor is a plan-construction error. Years within the active range with no stored
   point carry forward from the most recent prior anchor (carry-forward identity per
   `unit: Rate`). Years outside the active range are out of scope — the engine falls
   back to the plan-level stream or performs no lookup. Zero is never a valid in-scope
   allocation result.

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

10. Every `MemberLifecycleStream` must have an explicit anchor at `age = 0` with
    `phase = 0`. This is a plan-construction requirement enforced at load time.

11. A spousal relationship (`RelationSpouseStream`) requires exactly one other member
    with `MemberRole = Primary` or `MemberRole = Spouse` in the household. At
    most one spousal pair is permitted per plan.

12. `AccountBalanceStream` streams must have `start` year `>= account.opening_year`.

---

## Deferred: Retirement Budget

Income sources (Social Security benefits, pensions, wages) and expenses
(living, health, housing, discretionary) are not modeled as streams. They
are configuration inputs to the retirement budget computation, which determines
how much a household needs to withdraw from accounts each year. Entity schemas
for income and expense parameters, and how they feed into withdrawal planning,
will be specified when the retirement budget concept is designed.

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
| `assumptions / inflation / cpi` | — | `InflationCpiStream` |
| `assumptions / rates / large-cap` | — | `ReturnRateLargeCapStream` |
| `assumptions / rates / small-cap` | — | `ReturnRateSmallCapStream` |
| `assumptions / rates / international` | — | `ReturnRateInternationalStream` |
| `assumptions / rates / bonds` | — | `ReturnRateBondsStream` |
| `assumptions / rates / real-estate` | — | `ReturnRateRealEstateStream` |
| `assumptions / rates / vehicle` | — | `DepreciationRateVehicleStream` |
| `assumptions / allocations` | — | `AllocationStream` (owner=Plan) |
| `assumptions / allocations / [ member ] / [ account ]` | — | `AllocationStream` (owner=MemberAccount) |
| `assumptions / allocations / [ account ]` | — | `AllocationStream` (owner=JointAccount) |
| `assumptions / policy / limits / hsa` | — | `PolicyHsaLimitsStream` |
| `household / [ member ]` | `Member` | `MemberLifecycleStream` |
| `household / [ member ] / relations / spouse` | — | `RelationSpouseStream` |
| `assets / accounts / [ member ] / [ 401k ]` | `Account` (kind=Traditional401k) | `AccountBalanceStream` |
| `assets / accounts / [ member ] / [ roth-401k ]` | `Account` (kind=Roth401k) | `AccountBalanceStream` |
| `assets / accounts / [ member ] / [ ira ]` | `Account` (kind=TraditionalIra) | `AccountBalanceStream` |
| `assets / accounts / [ member ] / [ roth-ira ]` | `Account` (kind=RothIra) | `AccountBalanceStream` |
| `assets / accounts / [ member ] / [ hsa ]` | `Account` (kind=Hsa) | `AccountBalanceStream` |
| `assets / accounts / [ member ] / [ bank ]` | `Account` (kind=Bank, owner=MemberOwned) | `AccountBalanceStream` |
| `assets / accounts / [ member ] / [ brokerage ]` | `Account` (kind=Brokerage, owner=MemberOwned) | `AccountBalanceStream` |
| `assets / accounts / [ bank ]` | `Account` (kind=JointBank, owner=JointOwned) | `AccountBalanceStream` |
| `assets / accounts / [ brokerage ]` | `Account` (kind=JointBrokerage, owner=JointOwned) | `AccountBalanceStream` |
| `assets / hard-assets / properties / [ home ]` | `Property` (kind=PrimaryResidence) | `PropertyValueStream` |
| `assets / hard-assets / properties / [ vacation-home ]` | `Property` (kind=VacationHome) | `PropertyValueStream` |
| `assets / hard-assets / properties / [ parcel ]` | `Property` (kind=LandParcel) | `PropertyValueStream` |
| `assets / hard-assets / [ vehicle ]` | `Vehicle` | `VehicleValueStream` |
| `contributions / [ member ] / [ 401k ] / employee` | — | `Contribution401kEmployeeStream` |
| `contributions / [ member ] / [ 401k ] / employer` | — | `Contribution401kEmployerStream` |
| `contributions / [ member ] / [ roth-401k ] / employee` | — | `ContributionRoth401kEmployeeStream` |
| `contributions / [ member ] / [ roth-401k ] / employer` | — | `ContributionRoth401kEmployerStream` |
| `contributions / [ member ] / [ hsa ] / employee` | — | `ContributionHsaEmployeeStream` |
| `contributions / [ member ] / [ hsa ] / employer` | — | `ContributionHsaEmployerStream` |
| `contributions / [ member ] / [ ira ] / standard` | — | `ContributionIraStandardStream` |
| `contributions / [ member ] / [ roth-ira ] / standard` | — | `ContributionRothIraStandardStream` |
| `contributions / [ member ] / [ 401k ] / catchup` | — | `Contribution401kCatchupStream` |
| `contributions / [ member ] / [ roth-401k ] / catchup` | — | `ContributionRoth401kCatchupStream` |
| `contributions / [ member ] / [ ira ] / catchup` | — | `ContributionIraCatchupStream` |
| `contributions / [ member ] / [ roth-ira ] / catchup` | — | `ContributionRothIraCatchupStream` |
| `contributions / [ member ] / [ bank ]` | — | `ContributionBankStream` |
| `contributions / [ member ] / [ brokerage ]` | — | `ContributionBrokerageStream` |
| `contributions / [ bank ]` | — | `ContributionBankStream` (owner=JointAccount) |
| `contributions / [ brokerage ]` | — | `ContributionBrokerageStream` (owner=JointAccount) |
| `withdrawals / [ member ] / [ 401k ] / standard` | — | `Withdrawal401kStandardStream` |
| `withdrawals / [ member ] / [ roth-401k ]` | — | `WithdrawalRoth401kStream` |
| `withdrawals / [ member ] / [ hsa ] / qualified` | — | `WithdrawalHsaQualifiedStream` |
| `withdrawals / [ member ] / [ ira ] / standard` | — | `WithdrawalIraStandardStream` |
| `withdrawals / [ member ] / [ roth-ira ]` | — | `WithdrawalRothIraStream` |
| `withdrawals / [ member ] / [ bank ]` | — | `WithdrawalBankStream` |
| `withdrawals / [ member ] / [ brokerage ]` | — | `WithdrawalBrokerageStream` |
| `withdrawals / [ bank ]` | — | `WithdrawalBankStream` (owner=JointAccount) |
| `withdrawals / [ brokerage ]` | — | `WithdrawalBrokerageStream` (owner=JointAccount) |
| `liabilities / [ amortized-loan ]` | `AmortizedLoan` | `LoanBalanceStream`, `LoanPaymentStream` |
| `liabilities / [ interest-only-loan ]` | `InterestOnlyLoan` | `InterestOnlyBalanceStream`, `InterestOnlyPaymentStream` |
| `liabilities / [ credit-line ]` | `CreditLine` | `CreditLineBalanceStream`, `CreditLinePaymentStream` |
