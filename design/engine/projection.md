# Engine — Projection Algorithm

This document specifies how the projection engine evaluates the stream tree to
produce a `Projection`. It covers the three phases — timeline derivation,
projection tree construction, and the year sweep — and the per-stream evaluation
rules that apply during the sweep.

Stream structure, procedures, and formulas are defined in
[design/data-model.md](../data-model.md) and are referenced here, not repeated.
The `Model` facade contract (inputs, outputs, error handling) is specified in
[design/engine/api.md](api.md).

---

## Overview

`Model::get_projection(plan)` runs three sequential phases:

1. **Timeline derivation** — determine the year range to project.
2. **Tree construction** — build the ephemeral `Projection` stream tree from the plan.
3. **Year sweep** — evaluate every stream in the tree for every year in the timeline.

The result is a fully populated `Projection` whose streams and points are ready
for the CLI to render.

---

## Phase 1 — Timeline Derivation

The projection timeline is derived from all member lifetimes per
[data-model.md § Timeline Derivation](../data-model.md#timeline-derivation):

```
timeline_start = min(member.birth_year for all members in plan)
timeline_end   = max(member.birth_year + member.death_age for all members in plan)
```

These bounds are computed once and used as the outer loop bounds in Phase 3.
No stream configuration is read during this phase.

---

## Phase 2 — Projection Tree Construction

Before any values are computed, the engine constructs the ephemeral projection
stream tree. The tree structure is defined in
[data-model.md § Projection Output Streams](../data-model.md#projection-output-streams).

The engine creates:

- One `ProjectionAccountBalance` leaf stream per account in the plan, with
  `inputs` copied from the plan's `AccountBalance` `StreamTemplate` for that account.
  `inputs["rate"]` is the account's effective-rate `RateStream` root; all other
  inputs are the account's cash-flow `DollarStream`s.
- One `ProjectionHardAssetValue` leaf stream per property and vehicle.
  `inputs["rate"]` references the applicable return/depreciation `RateStream`.
  Capital improvement cash-flow streams are wired into `inputs` alongside `"rate"`.
- One `ProjectionLiabilityBalance` leaf stream per liability. The procedure
  matches the source liability kind:
  - `AmortizedLoan` → `AmortizedSchedule` (`inputs["rate"]`, `inputs["payment"]`, and any prepayment inputs)
  - `InterestOnlyLoan` → `InterestOnly` (`inputs["rate"]`)
  - `CreditLine` → `EndOfYearGrowth` (`inputs["rate"]` and any payment inputs)
- `Additive` intermediate aggregate streams: `retirement_aggregate`,
  `health_aggregate`, `taxable_aggregate`, `hard_assets_aggregate`,
  `long_term_aggregate`, `short_term_aggregate`, `assets`, `liabilities`.
- The `ProjectionNetWorth` root (`Additive`).

All `inputs` references are wired at construction time. The projection tree is
self-contained: evaluation proceeds top-down through `inputs` only.

Phase 2 produces a single `streams: Map<StreamId, Stream>` that merges
the plan's persisted streams with the newly created ephemeral projection
streams. This is the working set for Phase 3 — `eval` looks up every
stream from this one map. No separate ephemeral registry is needed.

**CPI stream resolution.** Before wiring any `EndOfYearGrowth` stream, Phase 2
resolves the plan's designated inflation stream once:

```
cpi_stream_id = the unique RateStream owned by Plan(plan_id) with label = "cpi".
```

Every `EndOfYearGrowth` procedure variant in the projection tree is constructed
with this `cpi_stream_id`. A plan must contain exactly one such stream; its
absence is a plan-construction error surfaced by `Model::load()` as
`InvalidPlan`. Substituting a different inflation stream (e.g., CPI-W for a
specific stream) requires setting a different `cpi_stream_id` in that procedure
instance — no other change is needed.

Projection streams are never added to the persisted `Plan` graph. They exist
only for the duration of `get_projection`.

---

## Phase 3 — Year Sweep

The sweep is an **outer-year, inner-depth-first** double loop:

```
for y in timeline_start..=timeline_end:
    1. Resolve all MemberLifecycleStream values for y.
    2. Evaluate the projection tree root: eval(projection_net_worth_id, y).
```

**Step 1** must precede step 2. Per
[data-model.md Design Invariant 9](../data-model.md#design-invariants),
`MemberLifecycleStream` values must be available before any stream that
depends on member age. This is the only fixed ordering constraint within a
year; all other evaluation order is determined recursively by the `inputs` map.

**Step 1 resolution** does not use the standard `eval` path. For each
`MemberLifecycleStream`, the engine computes:

- `age = y - member.birth_year` (derived, not carried forward from stored points)

The computed `age` is written to the memo table for year `y` so that downstream
streams can read it via `eval`.

`eval(stream_id, y)` is **memoized**: each `(stream_id, year)` pair is computed
at most once. When `eval` is called for a stream that has already been evaluated
for year `y`, the cached value is returned immediately. The memo table persists
across years — prior-year values must remain accessible for recurrence formulas
(`memo[stream_id][y-1]`) and cumulative CPI products (`eval(cpi, t)` for
`t < y`).

---

## Stream Evaluation — `eval(stream_id, y)`

```
eval(stream_id, y):
    if memo[stream_id][y] exists → return it

    stream = streams[stream_id]          -- unified map built in Phase 2

    if y < resolved_start(stream) or y > resolved_end(stream):
        result = identity_value(stream)
    else:
        stored     = stored_point_value(stream, y)
        input_vals = { name: eval(id, y) for (name, id) in stream.inputs }
        result     = dispatch(stream.procedure, stored, input_vals, stream_id, y)

    memo[stream_id][y] = result
    return result
```

`inputs` is the sole mechanism by which a stream references other streams.
All dependencies — rate references, cash-flow contributions, weighted-return
children, aggregate members — are named entries in the `inputs` map.

---

## Active Range and Identity Values

A stream is **active** for year `y` when:

```
resolved_start(stream) ≤ y ≤ resolved_end(stream)
```

`resolved_start` and `resolved_end` are derived per
[data-model.md § Stream Lifecycle](../data-model.md#stream-lifecycle):

- `StreamStart::CalendarYear(y)` resolves to `y`.
- `StreamStart::MemberAge(mid, age)` resolves to `member(mid).birth_year + age`.
- The end is derived from `terminates`, member death, entity lifecycle bounds,
  or `timeline_end` — in that priority order.

For years outside the active range, `eval` returns the **identity value** for
the stream's `ValueUnit` per
[data-model.md § Identity Semantics](../data-model.md#identity-semantics):

| Unit | Identity |
|------|----------|
| `Decimal` | `0` |
| `Rate`, `Count`, `Years`, `Age` | carry-forward from most recent `StreamPoint`; falls back to `AttributeDef.initial` |

Consumers of `eval` never branch on whether a stream is active — inactive
streams produce identity values that contribute nothing to any aggregate.

---

## Procedure Dispatch

### `Stored`

```
result = stored_point_value(stream, y)
```

Returns the carry-forward value of the most recent `StreamPoint` at or before
`y`, or the `initial` fallback. For `Decimal` streams with no stored point,
returns `0`. The `inputs` map is empty; no recursive evaluation occurs.

---

### `Product`

```
result = input_vals["weight"] × input_vals["rate"]
```

Used by `WeightedReturn` `RateStream`s in the effective rate tree.

---

### `Additive`

```
result = sum(input_vals.values())
```

Used by all aggregate streams — effective rate roots, projection sub-aggregates,
and the net worth root. `stored` is unused. Input slot names are user-defined labels.

---

### `EndOfYearGrowth`

The base case and the recurrence are handled separately.

**Base case** — when `y == resolved_start(stream)`:

```
(seed, seed_denom) = stored_point_value(stream, y)

if seed_denom is YZV:
    p(resolved_start) = yzv_to_ynv(seed, resolved_start, cpi)    -- assets
elif seed_denom is PNV(ref_year):
    p(resolved_start) = seed                                      -- liabilities (already nominal)
```

Asset seeds (accounts, hard assets) are stored in `YZV` and converted to
`YNV(resolved_start)`. Liability seeds (credit lines) are stored as
`PNV(start_year)` — nominal at the contract date — and used directly (see
[data-model.md § Liabilities](../data-model.md#liabilities)).

**Recurrence** — when `y > resolved_start(stream)`:

```
rate     = input_vals["rate"]
net_flow = sum(convert(k, v, y) for k, v in input_vals if k != "rate")
p(y)     = memo[stream_id][y-1] × (1 + rate) + net_flow
```

`p(y-1)` is available from the prior year's memo entry because the sweep
proceeds year by year. All cash-flow inputs are in `input_vals` under
user-defined keys; any key that is not `"rate"` is a cash-flow contribution.

**Denomination-aware conversion for net-flow inputs.** Each cash-flow input
is converted based on the denomination of its stored `StreamPoint`:

- `YZV` → `yzv_to_ynv(value, y, cpi)` — user-planned values in today's dollars
- `PNV(ref_year)` → used directly — historical actuals or nominal contractual values

```
yzv_to_ynv(value_yzv, y, cpi_stream_id) =
    value_yzv × ∏(1 + eval(cpi_stream_id, t))  for t = anchor_year, …, y−1
```

`cpi_stream_id` is the inflation stream id stored in the `EndOfYearGrowth`
procedure variant (resolved during Phase 2 — see CPI stream resolution above).
`eval(cpi_stream_id, t)` returns the carry-forward rate value for year `t`
via the normal evaluation path. This keeps CPI resolution within the memo
table and makes the inflation stream substitutable without changing the
procedure.

**Balance bound.** After the recurrence, the result is clamped toward zero
based on the stream's owner type:

```
-- asset streams (owner is Account, Property, or Vehicle): floor at zero
p(y) = max(p(y), 0.0)

-- liability streams (owner is Liability): ceiling at zero
p(y) = min(p(y), 0.0)
```

Without the ceiling, naively applying the recurrence past a loan's payoff
point (or over-payment of a credit line) would produce positive balances —
implying the lender owes the borrower money. The ceiling prevents that.

---

### `AmortizedSchedule`

The remaining balance is computed via a year-over-year recurrence that applies
`periods_per_year` payment periods of standard amortization to the prior
year-end balance. `periods_per_year` is stored on the procedure variant
(see [data-model.md § Stream Procedures](../data-model.md#stream-procedures)).

**Base case** — when `y == resolved_start(stream)`:

```
balance(resolved_start) = seed                           -- nominal principal (negative)
```

The seed is the loan's nominal principal, stored as `PNV(start_year)`. No
YZV→YNV conversion — liability values are nominal (see
[data-model.md § Liabilities](../data-model.md#liabilities)).

**Recurrence** — when `y > resolved_start(stream)`:

```
k         = periods_per_year
r_p       = input_vals["rate"] / k
pmt_p     = input_vals["payment"]                        -- nominal, not converted
extra     = sum(convert(key, v, y) for key, v in input_vals
                if key not in {"rate", "payment"})   -- denomination-aware per input

if r_p = 0:
    balance(y) = memo[stream_id][y-1] + pmt_p × k + extra
else:
    balance(y) = memo[stream_id][y-1] × (1 + r_p)^k
                 + pmt_p × ((1 + r_p)^k − 1) / r_p
                 + extra

balance(y) = min(balance(y), 0.0)                        -- ceiling at zero
```

The inner formula adds the payment term because `balance` is negative (liability
convention) and `pmt_p` is positive (payment toward the debt). Adding a positive
payment term to a negative balance moves the balance toward zero. This is the
standard annuity remaining-balance formula adapted for the negative-principal
sign convention, applied year-by-year rather than from inception.

`inputs["payment"]` is used as a nominal value (no YZV→YNV conversion) because
a fixed-rate loan payment is contractually fixed in nominal dollars. Extra
cash-flow inputs (prepayments, extra principal) ARE converted YZV→YNV, since
they represent user-planned actions expressed in today's purchasing power.

Mid-life modifications are handled naturally by the recurrence:
- **Rate change** (ARM, refinance): rate stream gets a new `StreamPoint`.
- **Re-amortization**: payment stream gets a new `StreamPoint`.
- **Prepayment / extra principal**: additional cash-flow input stream.
- **Early payoff**: ceiling at zero clamps the balance.

---

### `InterestOnly`

```
result = stored_point_value(stream, resolved_start(stream))   -- nominal opening balance
```

The principal does not change year over year. The stream emits the same
nominal opening balance for every active year. No iteration or denomination
conversion is needed — liability values are stored in nominal dollars (see
[data-model.md § Liabilities](../data-model.md#liabilities)), so the emitted
value is already in YNV(point.year) for each year.

Interest accrual is not emitted by this stream in MVP — it is a future concern
when expense streams are introduced.

---

## Sweep Output

After the sweep, the engine collects all computed values from the memo table
into `Projection.points`: a `Map<StreamId, Vec<StreamPoint>>` where each
`StreamPoint` carries `denomination = YNV(point.year)`.

`Projection.streams` contains all streams in the ephemeral projection tree.
The CLI navigates by filtering on `stream.label` and `stream.owner` per
[design/engine/api.md § Projection](api.md#projection).
