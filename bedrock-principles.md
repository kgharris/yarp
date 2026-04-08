# Bedrock Principles

Foundational principles governing all documentation and development work on this project.

---

## Separation of Concerns: Requirements, Design, and Implementation

Software development proceeds through distinct layers of decision-making. Each layer answers different questions:

- **Requirements** — What and Why. The problem, the behaviors, the constraints. Technology-agnostic.
- **Design** — How (structure). Architecture, data models, algorithms, interfaces. Technology-specific but implementation-independent.
- **Implementation** — How (code). Concrete libraries, exact signatures, coding conventions. Most volatile.

Each layer can change independently without affecting layers above it. Changing implementation doesn't change design. Changing design doesn't change requirements. But changing requirements often cascades downward.

**Key principle:** no layer restates or embeds content from another layer. Reference by link, never by duplication.

Each phase's scope — what belongs and what to exclude — is defined in [ai/workflows/context/](ai/workflows/context/).

---

## Model-View-Controller Architecture

This system is organized around the MVC pattern. Every component belongs to exactly one layer:

- **Model (Engine)** — data and computation. No awareness of how results are requested or displayed.
- **View (UX)** — presentation only. Receives shaped data from the Controller and renders it. Zero financial logic.
- **Controller** — mediates between Model and View. Receives requests, orchestrates computation, transforms output into view-ready payloads.

No layer reaches across another. The View never reads raw Model data directly. The Model never formats for display. The Controller never computes financial values.

---

## Data-Driven Business Rules

Business rules that are subject to external change — tax brackets, IRS contribution
limits, RMD divisor tables, Social Security parameters, IRMAA tiers, ACA thresholds,
estate tax exclusions — must be represented as data, not as logic.

Algorithms operate on rule data; they do not embed rule values. When a bracket
threshold changes, a data entry changes. No algorithm changes. When the IRS adjusts
contribution limits, a configuration value changes. No code changes. Requirements
define the structure and behavior of data (e.g., "a birth-year-to-start-age mapping
table"); they do not enumerate the specific values that populate it. Specific values
are shipped data, sourced from authoritative external publications and updatable
without changing requirements, design, or code.

**The test:** if a tax law changes, should any algorithm change? No — only data
should change. If the answer is yes, the rule is embedded in the wrong place.

---

## Stream-Based Computation

Every time-varying quantity in the data model is a stream. Account balances, asset
allocations, contribution limits, tax brackets, withdrawal strategies, income sources,
liabilities — all are streams that span the full timeline. Each stream yields a value
for every year, including a null result (which will have no impact on calculations) for
years where it is not yet active or has been depleted. Consumers never check whether a stream is active; they iterate and aggregate
what streams produce. Boundary awareness lives inside the streams, not in the code that
consumes them.

This pattern is recursive: each stream is itself composed of sub-streams that follow
the same rule. Boundary complexity is pushed into stream construction, not into the
computation that reads from streams.

Streams also capture user intent over time. When a user sets an asset allocation for
"10 years before retirement," that becomes a point in the allocation stream. The stream
carries the allocation forward until the next user-specified point, then transitions.
The projection logic that consumes allocations never checks transition rules — it reads
the stream value for each year.

Data-driven business rules and streams interrelate: externally-sourced rule data (tax
brackets, contribution limits, RMD tables) is delivered to the engine as streams. The
stream carries the rule data for each year; the algorithm consumes whatever the stream
produces without knowing whether the value came from a shipped default, a user override,
or an inflation-adjusted projection.

The streams model is the architectural foundation for how the system handles complex
time sequencing. Because streams uniformly span the full timeline and encode their own
boundary logic, the same stream infrastructure supports full projections, fast-forward
to a target year, subset views (e.g., a single account or a date range), and dollar-value
presentation modes — all without duplicating computation logic. Each consumption
pattern is a different way of iterating or slicing the same streams.

**The test:** if a new boundary condition requires adding an `if` statement to a
computation function, a stream is incomplete — fix the stream, not the consumer.

---

## Dollar-Value Denomination

Every dollar amount in the system exists in exactly one of four reference frames:

| Abbrev | Name | Reference Frame | Role |
|--------|------|-----------------|------|
| **YZV** | Year Zero Value | Plan anchor year (fixed) | Storage and computation anchor |
| **CNV** | Current Nominal Value | Current calendar year (slides forward) | Default user-facing display |
| **YNV** | Year Nominal Value | A specific year's nominal dollars (computed) | Projection output |
| **PNV** | Past Nominal Value | A specific historical year's dollars (observed) | Historical actuals |

No dollar amount is ever ambiguous. Every field, parameter, return value, and display
cell is denominated in exactly one frame. The denomination is part of the type, not a
comment — code that passes a dollar amount without its denomination is a bug.

**Storage is always YZV.** All persisted configuration and assumption values are stored
in year-zero dollars. YZV is fixed at the plan anchor year and does not change as the
calendar advances (unless the user explicitly rebases). This means stored values are
stable — they don't shift when the application is opened in a new year.

**Historicals are PNV.** Actuals data — year-end balances, IRS-published contribution
limits, quarterly balance sheet snapshots — is stored in the nominal dollars of its
reference date. PNV values are observed facts, not projections — they are invariant
once recorded. Each PNV value carries its reference year (the year the dollars are
denominated in). PNV is never silently converted to another denomination.

**Display defaults to CNV.** The user sees values in current-year purchasing power. The
CNV reference year is typically the current calendar year, but certain views may shift it
based on user selection — for example, viewing a future retirement budget may set CNV to
that budget year so the numbers feel tangible in that context. The UI converts YZV to CNV
using cumulative inflation (actual rates for elapsed years, projected rates for future
years). The user never enters or sees YZV directly unless they switch to the year-zero
display mode.

**Projections produce YNV.** YNV expresses a value in a specific year's nominal
dollars. Unlike PNV, YNV is always derived — it is computed by applying inflation
factors, not observed from external data. YNV is per-year — 2030 has its own YNV,
2040 has its own YNV. The target year can be in the past or future relative to CNV;
what makes it YNV (not PNV) is that the value is computed, not recorded. The primary
direction is forward: the engine inflates YZV values to nominal dollars for each
projection year. YNV can also project backwards from CNV to show what a value would be
in a past year's nominal dollars. Whether YNV should project backwards from YZV
(deflating below the anchor year) is an open design question — do not assume either
direction until this is resolved in the design layer.

**PNV vs YNV:** Both express dollars in a specific year's nominal terms. The
distinction is semantic: PNV is an observed actual (invariant once recorded), while
YNV is a derived projection (recomputed when assumptions change). A 2023 year-end
balance snapshot is PNV. A projected 2023 balance computed from the model is YNV —
even though both are denominated in 2023 dollars.

**Conversions are explicit and bidirectional:**
- YZV ↔ CNV: cumulative inflation factor from year-zero to current year
- YZV → YNV: cumulative inflation factor from year-zero to target year (forward projection)
- CNV → YNV: cumulative inflation factor from current year to target year (forward or backward)
- PNV ↔ YZV: cumulative inflation factor from PNV reference year to year-zero

**The test:** if a function accepts or returns a dollar amount and you cannot tell from
the type signature which denomination it is in, the interface is underspecified.

---

## Principles Trump Templates

These principles override any documentation template, pattern, or convention. If a document violates these principles, the violation stands regardless of other qualities.

When creating new documentation:
1. Identify which layer you're documenting
2. Write only content appropriate to that layer
3. Reference other layers by link, never by duplication
