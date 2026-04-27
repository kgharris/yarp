# CLI Design

> **MVP scope.** This CLI is both the MVP user interface and a scriptable test
> tool. It makes the engine directly accessible from CI pipelines and integration
> tests, and serves as the primary user-facing interface for the MVP release.
> The interface is intentionally minimal.

---

## Architecture Position

The CLI is a standalone Rust binary crate (`yarp-cli`) that links the engine and
the Model facade as library dependencies. It does not go through the Controller
or the Tauri IPC layer. It calls the Model facade to load or generate a plan, then
calls the engine directly and writes output to stdout. No UI state exists.

```
yarp-cli  →  Model  →  JsonPlanStore
               ↓
            Engine  →  stdout
```

The CLI never instantiates `JsonPlanStore` or any other persistence backend
directly. All persistence is accessed through the `Model` trait defined in
[design/architecture.md](../architecture.md#model-facade). This keeps the CLI
on the correct side of the Model boundary and allows the persistence backend to
be swapped (e.g., to `MemoryPlanStore` in tests) without changing the CLI.

---

## Invocation Modes

The CLI has exactly two mutually exclusive modes:

| Mode | When | Description |
|------|------|-------------|
| **Projection** | a positional `<dir>` argument is given (default) | Load plan, run projection, write table |
| **Generate** | `--generate-plan <dir>` is given | Write a minimal runnable plan directory |

Invoking with no arguments emits a usage summary to stdout and exits 0.

Supplying both a positional directory and `--generate-plan` in the same
invocation is an error (exit 1).

Supplying both `--json` and `--table` in the same invocation is an error
(exit 1).

---

## Command-Line Interface

```
yarp-cli <dir> [--json | --table]
yarp-cli --generate-plan <dir>
```

| Argument / Flag | Required | Description |
|-----------------|----------|-------------|
| `<dir>` | yes (projection mode) | Positional path to a directory of JSON plan files in `JsonPlanStore` format |
| `--json` | no | Emit JSON to stdout instead of the default CSV |
| `--table` | no | Emit an aligned text table to stdout instead of the default CSV |
| `--generate-plan <dir>` | yes (generate mode) | Write an initial plan directory to this path; directory must not already exist |

One positional argument. No subcommands.

---

## Projection Mode

### Error Display

The CLI does not define error types, codes, or messages. All errors originate
from `ModelError` and follow the format defined in
[design/error-handling.md](../error-handling.md). The CLI writes each
`ModelError`'s formatted string to stderr verbatim and exits 1.

### Validation

Validation is performed by `Model::load()`, not by the CLI. The CLI calls
`load()` and, on failure, writes the returned `ModelError` to stderr. The
validation checks — directory existence, plan structure, required assumptions,
allocation sums — are defined in the Model and DB layers. The CLI does not
duplicate or reinterpret them.

### Denomination

All monetary values are output in YNV (year nominal value) — each year's values
are denominated in that year's nominal dollars. No denomination conversion is
applied. No denomination toggle is provided.

### Data Shape

All output formats share the same orientation: **columns are years, rows are
streams**. The first column is a row label; subsequent columns are one per
projection year in ascending order.

Row labels use names directly from the plan data: streams with a label use
it (aggregates: `"retirement"`, `"net-worth"`); streams without a label use
the entity's display name via the owner (leaves: `"Alice 401k"`, `"Home"`;
lifecycle: `"Alice"`). No prefixes — the tree structure provides context.
Values containing commas are rejected at plan validation as `invalid-plan`
errors — row labels must be safe for CSV output without quoting.

Rows are emitted in **leaf-first tree order** by recursively walking the
projection tree from the root. For each stream, the walk emits all children
(via `inputs`) first, then emits the stream's own row. Children that are
not in the projection (rates, contributions) are skipped — the lookup
returns nothing and recursion stops naturally. Lifecycle rows are emitted
before the tree walk.

Rows emitted, in order:

| Row label | Content |
|-----------|---------|
| `<given_name>` | Age of the member in that year (one row per member) |
| `<label>` | End-of-year account balance (one row per retirement account) |
| `retirement` | Retirement aggregate (sum of 401k, Roth 401k, IRA, Roth IRA) |
| `<label>` | End-of-year account balance (one row per HSA) |
| `health` | Health aggregate (sum of HSA) |
| `<label>` | End-of-year account balance (one row per bank/brokerage) |
| `taxable` | Taxable aggregate (sum of bank and brokerage) |
| `<label>` | End-of-year hard asset value (one row per property or vehicle) |
| `hard-assets` | Hard assets aggregate (sum of properties and vehicles) |
| `assets` | Total assets (retirement + health + taxable + hard-assets) |
| `<label>` | End-of-year balance (one row per amortized or interest-only loan) |
| `lt-liabilities` | Long-term liabilities aggregate (amortized + interest-only loans) |
| `<label>` | End-of-year balance (one row per credit line) |
| `st-liabilities` | Short-term liabilities aggregate (credit lines) |
| `liabilities` | Total liabilities (long-term + short-term) |
| `net-worth` | Net worth (assets minus liabilities) |

### CSV Output (default)

RFC 4180 CSV written to stdout. First row is a header: `stream,<year>,<year>,...`.
One data row per stream. No quoting unless a field contains a comma. Monetary
values are bare numbers rounded to two decimal places — no currency symbols.
Age values are integers. Points with denomination `IDV` (identity values —
the stream is outside its active range) render as empty cells (no value
between the surrounding commas).

### Table Output (`--table`)

The same row/column orientation as CSV, formatted as a standard ASCII table.
Layout rules:

- Each column is fixed-width: the width of the widest value in that column
  (including the header), with a minimum of 4 characters for value columns
  (the width of a 4-digit year header).
- Columns are separated by two spaces.
- The label column is left-aligned. All value columns are right-aligned.
- The header row is followed by a separator row of `-` characters, one `-`
  per character position across the full row width (including inter-column spacing).
- No border characters on the left or right edges.
- No trailing whitespace on any line.
- Monetary values are bare numbers rounded to two decimal places — no currency
  symbols. Age values are integers. Same formatting rules as CSV.
- Points with denomination `IDV` render as blank cells (padded to column
  width with spaces).

Example (illustrative widths only):

```
stream                   2025      2026      2027
------------------------------------------------
Alice                      45        46        47
Alice 401k          123456.78 130000.00 137000.00
```

Intended for manual inspection; not suitable for programmatic parsing.

### JSON Output (`--json`)

A single JSON object written to stdout:

```json
{
  "denomination": "ynv",
  "years": [2025, 2026, ...],
  "rows": [
    { "stream": "<given_name>", "values": [45, 46, ...] },
    { "stream": "<label>", "values": [123456.78, 130000.00, ...] },
    ...
  ]
}
```

- `denomination` is always `"ynv"`. Each year's values are in that year's nominal dollars.
- `years` lists the projection years in ascending order.
- `rows` is an array of objects in the same row order as CSV. Each object has
  a `stream` label and a `values` array parallel to `years`. Points with
  denomination `IDV` render as `null` in the `values` array.
- Monetary values are JSON numbers (not strings), rounded to two decimal places
  in output only; internal computation remains full-precision. No currency symbols.
- Age values are JSON integers.

---

## Generate Mode

`--generate-plan <dir>` creates a minimal plan directory at the given path. If
the directory already exists, its plan files are overwritten. Filesystem
failures are returned as `ModelError` and written to stderr by the CLI
(exit 1).

The generated plan contains:

- Two household members:
  - Alice (`role = Primary`): `given_name = "Alice"`, `birth_year = current_year − 35`,
    `death_age = 90`, `retirement_age = 65`.
  - Bob (`role = Spouse`): `given_name = "Bob"`, `birth_year = current_year − 33`,
    `death_age = 90`, `retirement_age = 65`.
- A spousal relationship between Alice and Bob.
- One retirement lifecycle event per member at `age = 65`.
- One traditional 401k account per member (`label = "Alice 401k"` and `label = "Bob 401k"`)
  each with a starting balance.
- One employee 401k contribution `DollarStream` per member (inputs key
  `"employee-401k"` on the account balance stream) with
  `start = MemberAge(M.id, age: 22)` and `terminates = OnEvent(retirement_event_id)`.
- One employer 401k contribution `DollarStream` per member (inputs key
  `"employer-401k"` on the account balance stream) with
  `start = MemberAge(M.id, age: 22)` and `terminates = OnEvent(retirement_event_id)`.
- One joint bank account with `label = "Bank"` and a starting balance.
- One plan-level set of allocation weight `RateStream`s (label=`"weight.<class>"`)
  with a single anchor point each: placeholder percentages across large-cap,
  small-cap, international, and bonds summing to 100%.
- A full set of assumptions populated with placeholder values (CPI inflation rate,
  return rates for each asset class).

`anchor_year` is written as a top-level field in the generated plan and equals
the current calendar year at generation time. All monetary placeholder values
are YZV denominated in that `anchor_year`.

On success, the CLI writes exactly one line to stdout:

```
Plan created in <dir>
```

where `<dir>` is the path argument supplied by the user.

The generated files are in `JsonPlanStore` format — they are directly editable JSON.

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Any error (validation failure, conflicting flags, I/O error, plan load failure) |

Error messages go to stderr. Successful output goes to stdout. Nothing is
interleaved.
