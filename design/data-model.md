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

1. If the stream declares a `lifecycle-binding` field that corresponds to a terminal event → ends when the referenced
   lifecycle event fires (see [Lifecycle Events](#lifecycle-events) below).
2. Else if the stream is owned by a member (`OwnedBy::Member` or
   `OwnedBy::MemberAccount`) → ends at `member.birth_year + member.death_age`.
3. Else if the stream is owned by a bounded entity (e.g., `Account` with a
   `closing_year`, `Property` with a `sale_year`, `Business` with an
   `exit_year`) → ends when that entity field indicates, or extends to the plan
   timeline end if the field is `None`.
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

EventKind = Retirement | SocialSecurityClaiming | Custom
```

A `LifecycleEvent` with `kind = Retirement` fires at `member.birth_year +
member.retirement_age`. Its `age` field holds the retirement age directly —
changing `retirement_age` on the member means creating or updating the event,
which then cascades to all streams that reference it.

Streams that terminate at a `LifecycleEvent` set their `terminates` field to
`OnEvent { event_id }`. Streams that run until the member's natural death do
not set `terminates` at all.

### Stream

```
Stream {
    id:           Uuid,
    kind:         StreamKind,
    owner:        OwnedBy,
    start:        StreamStart,             -- when the stream becomes active
    lifecycle-bindings:   Option<[LifecycleEvent]>,  -- None = ends at natural bound (see Stream Lifecycle)
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
- `lifecycle-bindings` — if set, the stream ends when the referenced `LifecycleEvent`
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
    | Business(business_id: Uuid)
    | Income(income_id: Uuid)
    | Expense(expense_id: Uuid)
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
| `AccountBalance` | `Additive` — net cash flow from child `CashFlow` streams applied to the balance recurrence formula |
| `IncomeW2`, `IncomeSS`, etc. | `Additive` — sum of child income streams |
| `ExpenseLiving`, `ExpenseHousing`, etc. | `Additive` — sum of child expense streams |
| `AllocationGlobal`, `AllocationMemberAccount`, `AllocationJointAccount` | `Delegated` — each account resolves its effective allocation by checking account-level stream first, then falling back to plan-level global stream |
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
    | MemberLifecycle
    | RelationSpouse

    -- Accounts
    | AccountBalance

    -- Cash flows (contributions and withdrawals)
    | Contribution401kEmployee
    | Contribution401kEmployer
    | ContributionRoth401kEmployee
    | ContributionRoth401kEmployer
    | ContributionIraStandard
    | ContributionRothIraStandard
    | ContributionHsaEmployee
    | ContributionHsaEmployer
    | ContributionBank
    | ContributionBrokerage
    | Withdrawal401kStandard
    | WithdrawalRoth401k
    | WithdrawalIraStandard
    | WithdrawalRothIra
    | WithdrawalHsaQualified
    | WithdrawalBank
    | WithdrawalBrokerage

    -- Income
    | IncomeW2
    | IncomeSocialSecurity

    -- Expenses
    | ExpenseHousing
    | ExpenseHealthInsuranceEmployer
    | ExpenseHealthOutOfPocket
    | ExpenseLiving
    | ExpenseDiscretionary
    | ExpenseCustom

    -- Liabilities
    | LoanBalance
    | LoanPayment
    | InterestOnlyBalance
    | InterestOnlyPayment
    | CreditLineBalance
    | CreditLinePayment

    -- Hard assets
    | PropertyValue
    | VehicleValue

    -- Assumptions — inflation
    | InflationCpi

    -- Assumptions — return rates
    | ReturnRateLargeCap
    | ReturnRateSmallCap
    | ReturnRateInternational
    | ReturnRateBonds
    | ReturnRateRealEstate

    -- Assumptions — allocations
    | AllocationGlobal
    | AllocationMemberAccount
    | AllocationJointAccount

    -- Policy — Social Security
    | PolicySsTaxationThresholds

    -- Policy — federal tax
    | PolicyFederalTaxBrackets
    | PolicyFederalStandardDeduction

    -- Projection outputs (computed; not stored inputs)
    | ProjectionAccountBalance
    | ProjectionTaxableIncome
    | ProjectionFederalTaxLiability
    | ProjectionSsTaxableAmount
    | ProjectionSurplusDeficit
```

**Note on `AccountType`:** The `AccountBalance` stream kind is common to all
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

RelationKind = Spouse

PropertyKind = PrimaryResidence | VacationHome | LandParcel

LiabilityKind = AmortizedLoan | InterestOnlyLoan | CreditLine

InflationRef = Cpi | None
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
does not change once set without an explicit rebase operation.

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
does not vary by year. The `MemberLifecycle` stream carries the member's active
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
    ss_claiming_age: Option<i32>,    -- None if member has no SS income
}
```

`birth_year` is the member's single calendar anchor. Every derived calendar
event — retirement, SS claiming — is computed as `birth_year + age`. Storing
ages rather than calendar years means that correcting `birth_year` automatically
shifts all downstream events.

No age-derived calendar year (i.e., `member.birth_year + age`) may be stored as
a raw `start_year` or `end_year` on a stream record. Age-relative starts are
expressed as `StreamStart::MemberAge`. Age-relative ends are expressed via
`TerminationRef::OnEvent`, where the event's `age` field is the authoritative
value.

**MemberLifecycle stream:**

```
MemberLifecycle stream for member M:
    kind         = MemberLifecycle
    owner        = Member(M.id)
    start        = MemberAge(M.id, age: 0)
    -- no terminates — MemberLifecycle ends at member death, the natural bound
    --   derived from Member.death_age
    value_schema = AttributeMap([
        { key: "age",   unit: Age   },
        { key: "phase", unit: Count },  -- 0=working, 1=retired (dimensionless enum)
    ])
```

The `phase` attribute transitions from `working` to `retired` at
`birth_year + retirement_age`. Consumers read the attribute from the stream;
no branching on age in consumer code.

**Design invariant — resolution order:** `MemberLifecycle` streams are resolved
before all streams that reference them (income streams, contribution bounds,
health insurance phase transitions). This is a named engine contract: the
engine's projection pass resolves household lifecycle streams first in each year
before evaluating any stream that depends on a member's phase or age attribute.

**Member relation streams:**

```
RelationSpouse stream:
    kind   = RelationSpouse
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
}
```

### Account Balance Stream

Each account has exactly one `AccountBalance` stream. This stream is the root
of all cash flow child streams for the account.

```
AccountBalance stream for account A:
    kind         = AccountBalance
    owner        = Account(A.id)
    start        = CalendarYear(A.opening_year)
    -- end derived from Account.closing_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- Stored input points (user-entered seed balances) are YZV.
    -- Projection output points for this account are separate:
    --   see ProjectionAccountBalance in Projection Output Streams below.
```

**Denomination separation:** The `AccountBalance` stream holds user-entered
seed values in `YZV`. Observed historical year-end snapshots are stored in
a separate `HistoricalBalance` record (denomination `PNV(reference_year)`).
Projected year-end balances are written by the engine to a separate
`ProjectionAccountBalance` stream (denomination `YNV(year)`). A single stream
does not carry two denominations.

```
HistoricalBalance {
    account_id:      Uuid,
    reference_year:  i32,
    balance:         Decimal,   -- [PNV(reference_year)]
}
```

The engine uses the most recent `HistoricalBalance` as the seed for projection.
`HistoricalBalance` records are immutable once recorded.

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
    appreciation_rate: Decimal,     -- dimensionless; annual rate
}
```

```
PropertyValue stream for property P:
    kind         = PropertyValue
    owner        = Property(P.id)
    start        = CalendarYear(P.purchase_year)
    -- end derived from Property.sale_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] seed; projected YNV per year by engine
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
    depreciation_rate: Decimal,     -- dimensionless; annual rate
}
```

```
VehicleValue stream for vehicle V:
    kind         = VehicleValue
    owner        = Vehicle(V.id)
    start        = CalendarYear(V.purchase_year)
    -- end derived from Vehicle.disposal_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] seed; projected YNV per year
```

---

## Contributions

Contribution streams are owned by accounts. Each contribution type is a
distinct stream kind. Accounts relate to contribution streams via a
one-to-many relationship keyed on `(account_id, kind)`. The engine resolves
contributions for an account by querying all contribution streams owned by
that account.

### 401k Contributions

```
Contribution401kEmployee stream for member M on account A:
    kind         = Contribution401kEmployee
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    start        = MemberAge(M.id, age: hire_age)
    terminates   = OnEvent(retirement_event_id)
    -- retirement_event_id references LifecycleEvent { kind: Retirement, age: M.retirement_age }
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; positive = inflow

Contribution401kEmployer stream:
    kind         = Contribution401kEmployer
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    start        = MemberAge(M.id, age: hire_age)
    terminates   = OnEvent(retirement_event_id)
    -- user-supplied annual amount; [YZV]
```

### Roth 401k Contributions

```
ContributionRoth401kEmployee stream:
    kind         = ContributionRoth401kEmployee
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    start        = MemberAge(M.id, age: hire_age)
    terminates   = OnEvent(retirement_event_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]

ContributionRoth401kEmployer stream:
    kind         = ContributionRoth401kEmployer
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    start        = MemberAge(M.id, age: hire_age)
    terminates   = OnEvent(retirement_event_id)
    -- user-supplied annual amount; [YZV]
```

### IRA Contributions

```
ContributionIraStandard stream:
    kind         = ContributionIraStandard
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    start        = MemberAge(M.id, age: 0)
    -- no terminates — open-ended; ends at member death (natural bound)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]
```

### Roth IRA Contributions

```
ContributionRothIraStandard stream:
    kind         = ContributionRothIraStandard
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    start        = MemberAge(M.id, age: 0)
    -- no terminates — open-ended; ends at member death (natural bound)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]
```

### HSA Contributions

```
ContributionHsaEmployee stream:
    kind         = ContributionHsaEmployee
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    -- active while member has HDHP coverage (working phase)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]

ContributionHsaEmployer stream:
    kind         = ContributionHsaEmployer
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    -- user-supplied annual amount; [YZV]
```

### Bank and Brokerage Contributions

```
ContributionBank and ContributionBrokerage:
    -- owner discriminates MemberAccount from JointAccount
    -- denomination = YZV; value = annual deposit amount
```

---

## Withdrawals

Withdrawal streams are owned by accounts, parallel to contributions. The engine
resolves withdrawals for an account by querying all withdrawal streams for that
account.

### 401k Withdrawals

```
Withdrawal401kStandard stream:
    kind         = Withdrawal401kStandard
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    -- voluntary; user-configurable amount per year
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; negative = outflow
```

### Roth 401k Withdrawals

```
WithdrawalRoth401k stream:
    kind         = WithdrawalRoth401k
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    -- qualified withdrawals; tax-free
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]
```

### IRA Withdrawals

```
WithdrawalIraStandard stream:
    kind         = WithdrawalIraStandard
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    -- voluntary; user-configurable amount per year
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]
```

### Roth IRA Withdrawals

```
WithdrawalRothIra stream:
    kind         = WithdrawalRothIra
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    -- qualified withdrawals; tax-free
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]
```

### HSA Withdrawals

```
WithdrawalHsaQualified stream:
    kind         = WithdrawalHsaQualified
    owner        = MemberAccount(M.id, A.id)
    parent_id    = A's AccountBalance stream id
    -- matched against eligible health expenses; tax-free
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]
```

### Bank and Brokerage Withdrawals

```
WithdrawalBank and WithdrawalBrokerage:
    -- owner discriminates MemberAccount from JointAccount
    -- denomination = YZV
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
LoanBalance stream:
    kind         = LoanBalance
    owner        = Liability(AmortizedLoan.id)
    start        = CalendarYear(start_year)
    -- end derived from AmortizedLoan.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] remaining principal per year

LoanPayment stream:
    kind         = LoanPayment
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
InterestOnlyBalance stream:
    kind         = InterestOnlyBalance
    owner        = Liability(InterestOnlyLoan.id)
    start        = CalendarYear(start_year)
    -- end derived from InterestOnlyLoan.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]

InterestOnlyPayment stream:
    kind         = InterestOnlyPayment
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
CreditLineBalance stream:
    kind         = CreditLineBalance
    start        = CalendarYear(start_year)
    -- end derived from CreditLine.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] revolving balance per year

CreditLinePayment stream:
    kind         = CreditLinePayment
    start        = CalendarYear(start_year)
    -- end derived from CreditLine.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] annual minimum or configured payment
```

---

## Income

Each income source is an anchor entity. The income stream it owns carries the
annual income amount.

### W-2 Income

```
W2Income {
    id:          Uuid,
    plan_id:     Uuid,
    member_id:   Uuid,
    employer:    string,
    start_age:   i32,       -- age at which this employment begins
    salary_yzv:  Decimal,   -- [YZV] base salary
    growth_rate: Decimal,   -- dimensionless; annual salary growth assumption
}
```

```
IncomeW2 stream:
    kind         = IncomeW2
    owner        = Income(W2Income.id)
    start        = MemberAge(member_id, age: W2Income.start_age)
    terminates   = OnEvent(retirement_event_id)
    -- retirement_event_id references LifecycleEvent { kind: Retirement, age: M.retirement_age }
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] base; inflated to YNV by engine
```

### Social Security Income

```
SocialSecurityIncome {
    id:           Uuid,
    plan_id:      Uuid,
    member_id:    Uuid,
    benefit_yzv:  Decimal,   -- [YZV] pre-computed annual benefit at claiming_age
    claiming_age: i32,       -- age at which benefits begin
}
```

`benefit_yzv` is the user-supplied annual benefit amount, already adjusted for
early or delayed claiming. The engine applies CPI inflation to project it to
`YNV` for each year.

```
IncomeSocialSecurity stream:
    kind         = IncomeSocialSecurity
    owner        = Income(SocialSecurityIncome.id)
    start        = MemberAge(member_id, age: claiming_age)
    -- no terminates — ends at member death (natural bound)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV] base; inflated to YNV by engine
```

---

## Expenses

Each expense entity holds the base amount and metadata. Time-varying behavior
is delegated to the stream.

### Housing Expenses

```
HousingExpense {
    id:         Uuid,
    plan_id:    Uuid,
    label:      string,          -- e.g., "Property tax", "HOA", "Maintenance"
    start_year: i32,
    end_year:   Option<i32>,
    amount_yzv: Decimal,         -- [YZV] annual housing expense
}
```

```
ExpenseHousing stream:
    kind         = ExpenseHousing
    owner        = Expense(HousingExpense.id)
    start        = CalendarYear(start_year)
    -- end derived from HousingExpense.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; inflates at CPI
    -- Excludes mortgage P&I (those are liability streams)
```

### Health Insurance — Employer-Sponsored

```
EmployerHealthInsurance {
    id:         Uuid,
    plan_id:    Uuid,
    member_id:  Uuid,
    hire_age:   i32,
    amount_yzv: Decimal,    -- [YZV] annual premium
}
```

```
ExpenseHealthInsuranceEmployer stream:
    kind         = ExpenseHealthInsuranceEmployer
    owner        = Expense(EmployerHealthInsurance.id)
    start        = MemberAge(member_id, age: hire_age)
    terminates   = OnEvent(retirement_event_id)
    -- retirement_event_id references LifecycleEvent { kind: Retirement, age: M.retirement_age }
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; inflates at CPI
```

### Health — Out-of-Pocket

```
HealthOutOfPocket {
    id:         Uuid,
    plan_id:    Uuid,
    start_year: i32,
    end_year:   Option<i32>,
    amount_yzv: Decimal,    -- [YZV] annual OOP medical costs
}
```

```
ExpenseHealthOutOfPocket stream:
    kind         = ExpenseHealthOutOfPocket
    owner        = Expense(HealthOutOfPocket.id)
    start        = CalendarYear(start_year)
    -- end derived from HealthOutOfPocket.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; inflates at CPI
```

### Living Expenses

```
LivingExpense {
    id:         Uuid,
    plan_id:    Uuid,
    start_year: i32,
    end_year:   Option<i32>,
    amount_yzv: Decimal,    -- [YZV] annual living expenses
}
```

```
ExpenseLiving stream:
    kind         = ExpenseLiving
    owner        = Expense(LivingExpense.id)
    start        = CalendarYear(start_year)
    -- end derived from LivingExpense.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; inflates at CPI
```

### Discretionary Expenses

```
DiscretionaryExpense {
    id:         Uuid,
    plan_id:    Uuid,
    label:      string,
    start_year: i32,
    end_year:   Option<i32>,
    amount_yzv: Decimal,    -- [YZV] annual discretionary spending
}
```

```
ExpenseDiscretionary stream:
    kind         = ExpenseDiscretionary
    owner        = Expense(DiscretionaryExpense.id)
    start        = CalendarYear(start_year)
    -- end derived from DiscretionaryExpense.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]; inflates at CPI unless overridden
```

### Custom Expenses

```
CustomExpense {
    id:            Uuid,
    plan_id:       Uuid,
    label:         string,
    inflation_ref: InflationRef,
    start_year:    i32,
    end_year:      Option<i32>,
    amount_yzv:    Decimal,     -- [YZV] annual expense
}
```

```
ExpenseCustom stream:
    kind         = ExpenseCustom
    owner        = Expense(CustomExpense.id)
    start        = CalendarYear(start_year)
    -- end derived from CustomExpense.end_year per stream lifecycle rule 3
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YZV]
    -- inflation_ref on the entity determines which inflation stream applies
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
InflationCpi stream:
    kind         = InflationCpi
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([{ key: "value", unit: Rate }])   -- dimensionless; e.g., 0.03
```

A single CPI rate is applied uniformly to all streams that inflate over time.

### Return Rate Assumptions

One stream per asset class; all have the same structure:

```
ReturnRateLargeCap stream:
    kind         = ReturnRateLargeCap
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([{ key: "value", unit: Rate }])   -- dimensionless; annual expected return

-- ReturnRateSmallCap, ReturnRateInternational, ReturnRateBonds,
-- ReturnRateRealEstate follow the same structure.
```

### Allocation Assumptions

Allocation streams are multi-attribute. Each point carries a named percentage
per asset class. The sum of all allocation attribute values for any active point
must equal 1.0 — this is a single-point validation, not a cross-stream constraint.

```
AllocationGlobal stream (plan-level default):
    kind         = AllocationGlobal
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([
        { key: "large_cap",     unit: Decimal },   -- fraction 0.0–1.0
        { key: "small_cap",     unit: Decimal },
        { key: "international", unit: Decimal },
        { key: "bonds",         unit: Decimal },
        { key: "real_estate",   unit: Decimal },
    ])
    -- Invariant: sum of all attribute values = 1.0 for each active point

AllocationMemberAccount stream (per-account override for member-owned accounts):
    kind         = AllocationMemberAccount
    owner        = Account(account_id)
    start        = MemberAge(member_id, age: override_start_age)
    -- "10 years before retirement" is encoded as an age, not a derived calendar year
    -- MemberAge start means that changing birth_year shifts the override automatically
    -- end derived from member's natural bound (rule 2) or via terminates if earlier

AllocationJointAccount stream (per-account override for joint accounts):
    kind         = AllocationJointAccount
    owner        = Account(account_id)
    start        = CalendarYear(override_start_year)
    -- end derived from plan timeline (rule 4)
```

The engine resolves the effective allocation for an account by checking for an
account-level override stream first, then falling back to the plan-level
`AllocationGlobal` stream.

---

## Policy Data

Policy streams carry externally-sourced regulatory values. They are stored as
streams so that shipped defaults populate current and near-future years, and
per-year overrides let users model legislative changes without code changes.
The engine consumes them identically to user assumption streams.

All policy streams are owned by the plan. Historical values for past years are
`PNV(ref_year)`. Future-year defaults shipped with the application are `YZV`,
inflated to each target year (`YNV`) at runtime.

### Policy Streams — Social Security

```
PolicySsTaxationThresholds stream:
    kind         = PolicySsTaxationThresholds
    owner        = Plan(plan_id)
    start        = CalendarYear(plan.anchor_year)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([
        { key: "floor_mfj",      unit: Decimal },  -- [YZV] lower combined-income threshold MFJ
        { key: "ceiling_mfj",    unit: Decimal },  -- [YZV] upper combined-income threshold MFJ
        { key: "floor_single",   unit: Decimal },  -- [YZV] lower threshold for single filers
        { key: "ceiling_single", unit: Decimal },  -- [YZV] upper threshold for single filers
    ])
    -- Not inflation-indexed; values are PNV for historical years, stable forward
```

### Policy Streams — Federal Tax

#### Federal Income Tax Brackets

```
FederalTaxBracketsTable {
    id:             Uuid,
    entries_mfj:    [TaxBracket],
    entries_single: [TaxBracket],
}

TaxBracket {
    rate:            Decimal,   -- dimensionless; marginal rate (e.g., 0.22)
    threshold_floor: Decimal,   -- [YZV] income above which this rate applies
}
```

```
PolicyFederalTaxBrackets stream:
    kind         = PolicyFederalTaxBrackets
    owner        = Plan(plan_id)
    start        = CalendarYear(first year of published brackets)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- value = PolicyTableId reference to FederalTaxBracketsTable
    -- Per-year override supported; carry-forward semantics apply
```

#### Federal Standard Deduction

```
PolicyFederalStandardDeduction stream:
    kind         = PolicyFederalStandardDeduction
    owner        = Plan(plan_id)
    start        = CalendarYear(first year of published deduction)
    -- end derived from plan timeline (rule 4)
    value_schema = AttributeMap([
        { key: "mfj",    unit: Decimal },   -- [YZV]
        { key: "single", unit: Decimal },   -- [YZV]
    ])
    -- Inflates annually at CPI unless point is explicitly overridden
```

---

## Projection Output Streams

The projection engine writes its results as streams. Output streams are computed,
not stored as inputs. All output stream points carry denomination `YNV(point.year)`.

```
ProjectionAccountBalance stream (per account):
    kind         = ProjectionAccountBalance
    owner        = Account(account_id)
    start        = CalendarYear(account.opening_year)
    -- end derived from plan timeline end (rule 4)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- Each point: denomination = YNV(point.year)
    -- Distinct from AccountBalance stream; that stream holds YZV seed inputs.
    -- Account balance floor = 0; engine enforces before writing each point.
```

```
ProjectionTaxableIncome stream (per plan):
    kind         = ProjectionTaxableIncome
    owner        = Plan(plan_id)
    value_schema = AttributeMap([
        { key: "gross_income",       unit: Decimal },  -- [YNV(year)]
        { key: "standard_deduction", unit: Decimal },  -- [YNV(year)]
        { key: "taxable_income",     unit: Decimal },  -- [YNV(year)]
    ])

ProjectionFederalTaxLiability stream (per plan):
    kind         = ProjectionFederalTaxLiability
    owner        = Plan(plan_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YNV(year)] federal income tax owed

ProjectionSsTaxableAmount stream (per member):
    kind         = ProjectionSsTaxableAmount
    owner        = Member(member_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])   -- [YNV(year)] taxable portion of SS benefit

ProjectionSurplusDeficit stream (per plan):
    kind         = ProjectionSurplusDeficit
    owner        = Plan(plan_id)
    value_schema = AttributeMap([{ key: "value", unit: Decimal }])
    -- [YNV(year)] after-tax income minus total expenses; positive = surplus
```

**Balance recurrence formula** (end-of-year convention per
[D:formulas / balance-recurrence / end-of-year](../requirements/conceptual-model.md#d-formulas)):

```
p(y) = p(y-1) * (1 + effective_rate(y)) + net_flow(y)
```

where `effective_rate(y)` is the weighted average of return rates using the
resolved allocation stream for that account and year, and `net_flow(y)` is the
sum of all contribution and withdrawal stream values for that year, converted
from `YZV` to `YNV(y)` before the recurrence is applied.

---

## Design Invariants

These invariants are enforced at plan construction time (fail-fast at
boundaries), not deferred to projection.

1. All `[YZV]`-denominated entity fields contain values denominated in
   `anchor_year` dollars. No `CNV` or `YNV` values may be stored in input
   entity fields.

2. `CNV` never appears on any stored `StreamPoint`. CNV is a display-layer
   denomination produced by the controller.

3. For projection output stream points, `denomination = YNV(point.year)`.
   The denomination of any stored point is determinable from the point record
   alone.

4. All allocation streams (`AllocationGlobal`, `AllocationMemberAccount`,
   `AllocationJointAccount`) must have attribute values summing to 1.0 for
   every active point. Validation is per-point — not a cross-stream constraint.

5. Every `MemberAge` start references a `member_id` that exists in the plan's
   household. Dangling member references are invalid.

6. `Member.birth_year` is immutable after plan creation without an explicit
   rebase operation. Changing `birth_year` cascades to all resolved calendar
   years for that member's streams — this is the mechanism by which age-based
   events stay coherent.

7. No age-derived calendar year (i.e., `birth_year + age`) may be stored as a
   raw `start_year` or `end_year` on a stream record. Age-relative starts are
   expressed as `StreamStart::MemberAge`. Age-relative ends are expressed via
   `TerminationRef::OnEvent`.

8. No stream record stores an explicit end year or end age. Stream ends are
   derived from the ownership chain per the Stream Lifecycle rules.

9. `MemberLifecycle` streams are resolved before all streams that reference
   them in each projection year. This is the canonical engine resolution order.

10. `HistoricalBalance` records are immutable once recorded. `PNV` values are
    observed facts; they are never converted to another denomination silently.

11. A spousal relationship (`RelationSpouse`) requires exactly one other member
    with `MemberRole = Primary` or `MemberRole = Spouse` in the household. At
    most one spousal pair is permitted per plan.

12. `AccountBalance` streams must have `start` year `>= account.opening_year`.

---

## Open Design Questions

These questions do not block MVP but require decisions before the affected
features are implemented.

- **SS benefit currency:** `SocialSecurityIncome.benefit_yzv` is user-supplied
  and pre-adjusted for claiming age. The engine inflates it via `InflationCpi`
  each year. If a future phase restores FRA-based benefit derivation, the entity
  will need `pia_yzv` reinstated and `SsFraTable` introduced. Until then,
  `benefit_yzv` is the sole input and claiming-age adjustment is the user's
  responsibility.

- **YNV backward projection from YZV:** Bedrock principles note this is an open
  design question. The data model does not assume backward projection is supported.
  If added, projection output streams will need a backward-sweep variant and the
  denomination rules will need to address deflation below the anchor year.

---

## Cross-Reference Index

This table maps every `S:` path from the conceptual model to its design record
in this document. Every path from all sections (Assumptions, Household, Assets,
Contributions, Withdrawals, Liabilities, Income, Expenses) appears exactly once.

| Conceptual Model Path | Anchor Record | StreamKind |
|-----------------------|---------------|------------|
| `assumptions / inflation / cpi` | — | `InflationCpi` |
| `assumptions / rates / large-cap` | — | `ReturnRateLargeCap` |
| `assumptions / rates / small-cap` | — | `ReturnRateSmallCap` |
| `assumptions / rates / international` | — | `ReturnRateInternational` |
| `assumptions / rates / bonds` | — | `ReturnRateBonds` |
| `assumptions / rates / real-estate` | — | `ReturnRateRealEstate` |
| `assumptions / allocations` | — | `AllocationGlobal` |
| `assumptions / allocations / [ member ] / [ account ]` | — | `AllocationMemberAccount` |
| `assumptions / allocations / [ account ]` | — | `AllocationJointAccount` |
| `assumptions / policy / ss / taxation-thresholds` | — | `PolicySsTaxationThresholds` |
| `assumptions / policy / tax / federal / brackets` | `FederalTaxBracketsTable` | `PolicyFederalTaxBrackets` |
| `assumptions / policy / tax / federal / standard-deduction` | — | `PolicyFederalStandardDeduction` |
| `household / [ member ]` | `Member` | `MemberLifecycle` |
| `household / [ member ] / relations / spouse` | — | `RelationSpouse` |
| `assets / accounts / [ member ] / [ 401k ]` | `Account` (kind=Traditional401k) | `AccountBalance` |
| `assets / accounts / [ member ] / [ roth-401k ]` | `Account` (kind=Roth401k) | `AccountBalance` |
| `assets / accounts / [ member ] / [ ira ]` | `Account` (kind=TraditionalIra) | `AccountBalance` |
| `assets / accounts / [ member ] / [ roth-ira ]` | `Account` (kind=RothIra) | `AccountBalance` |
| `assets / accounts / [ member ] / [ hsa ]` | `Account` (kind=Hsa) | `AccountBalance` |
| `assets / accounts / [ member ] / [ bank ]` | `Account` (kind=Bank, owner=MemberOwned) | `AccountBalance` |
| `assets / accounts / [ member ] / [ brokerage ]` | `Account` (kind=Brokerage, owner=MemberOwned) | `AccountBalance` |
| `assets / accounts / [ bank ]` | `Account` (kind=JointBank, owner=JointOwned) | `AccountBalance` |
| `assets / accounts / [ brokerage ]` | `Account` (kind=JointBrokerage, owner=JointOwned) | `AccountBalance` |
| `assets / hard-assets / properties / [ home ]` | `Property` (kind=PrimaryResidence) | `PropertyValue` |
| `assets / hard-assets / properties / [ vacation-home ]` | `Property` (kind=VacationHome) | `PropertyValue` |
| `assets / hard-assets / properties / [ parcel ]` | `Property` (kind=LandParcel) | `PropertyValue` |
| `assets / hard-assets / [ vehicle ]` | `Vehicle` | `VehicleValue` |
| `contributions / [ member ] / [ 401k ] / employee` | — | `Contribution401kEmployee` |
| `contributions / [ member ] / [ 401k ] / employer` | — | `Contribution401kEmployer` |
| `contributions / [ member ] / [ roth-401k ] / employee` | — | `ContributionRoth401kEmployee` |
| `contributions / [ member ] / [ roth-401k ] / employer` | — | `ContributionRoth401kEmployer` |
| `contributions / [ member ] / [ hsa ] / employee` | — | `ContributionHsaEmployee` |
| `contributions / [ member ] / [ hsa ] / employer` | — | `ContributionHsaEmployer` |
| `contributions / [ member ] / [ ira ] / standard` | — | `ContributionIraStandard` |
| `contributions / [ member ] / [ roth-ira ] / standard` | — | `ContributionRothIraStandard` |
| `contributions / [ member ] / [ bank ]` | — | `ContributionBank` |
| `contributions / [ member ] / [ brokerage ]` | — | `ContributionBrokerage` |
| `contributions / [ bank ]` | — | `ContributionBank` (owner=JointAccount) |
| `contributions / [ brokerage ]` | — | `ContributionBrokerage` (owner=JointAccount) |
| `withdrawals / [ member ] / [ 401k ] / standard` | — | `Withdrawal401kStandard` |
| `withdrawals / [ member ] / [ roth-401k ]` | — | `WithdrawalRoth401k` |
| `withdrawals / [ member ] / [ hsa ] / qualified` | — | `WithdrawalHsaQualified` |
| `withdrawals / [ member ] / [ ira ] / standard` | — | `WithdrawalIraStandard` |
| `withdrawals / [ member ] / [ roth-ira ]` | — | `WithdrawalRothIra` |
| `withdrawals / [ member ] / [ bank ]` | — | `WithdrawalBank` |
| `withdrawals / [ member ] / [ brokerage ]` | — | `WithdrawalBrokerage` |
| `withdrawals / [ bank ]` | — | `WithdrawalBank` (owner=JointAccount) |
| `withdrawals / [ brokerage ]` | — | `WithdrawalBrokerage` (owner=JointAccount) |
| `liabilities / [ amortized-loan ]` | `AmortizedLoan` | `LoanBalance`, `LoanPayment` |
| `liabilities / [ interest-only-loan ]` | `InterestOnlyLoan` | `InterestOnlyBalance`, `InterestOnlyPayment` |
| `liabilities / [ credit-line ]` | `CreditLine` | `CreditLineBalance`, `CreditLinePayment` |
| `income / w2 / [ member ] / [ employer ]` | `W2Income` | `IncomeW2` |
| `income / ss / [ member ]` | `SocialSecurityIncome` | `IncomeSocialSecurity` |
| `expenses / housing` | `HousingExpense` | `ExpenseHousing` |
| `expenses / health / insurance / employer` | `EmployerHealthInsurance` | `ExpenseHealthInsuranceEmployer` |
| `expenses / health / out-of-pocket` | `HealthOutOfPocket` | `ExpenseHealthOutOfPocket` |
| `expenses / living` | `LivingExpense` | `ExpenseLiving` |
| `expenses / discretionary` | `DiscretionaryExpense` | `ExpenseDiscretionary` |
| `expenses / [ custom ]` | `CustomExpense` | `ExpenseCustom` |
