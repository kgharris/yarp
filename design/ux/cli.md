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
invocation is an error (exit 1, message to stderr):
`error: invalid-argument: --generate-plan and a positional directory are mutually exclusive`

Supplying both `--json` and `--table` in the same invocation is an error
(exit 1, message to stderr):
`error: invalid-argument: --json and --table are mutually exclusive`

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

### Error Format

All errors written to stderr follow this format:

```
<severity>: <type>: <variable-message>
```

- **severity** — `error` for all fatal conditions in MVP.
- **type** — a short kebab-case noun identifying the error class (see table below).
- **variable-message** — for system errors (I/O, permissions, path not found),
  this is the OS-level error message verbatim. For validation errors, it is a
  structured description defined per error class.

| Type | Variable message |
|---|---|
| `invalid-argument` | Human-readable description of the argument conflict |
| `system-error` | OS error message (e.g., `No such file or directory`, `Permission denied`) |
| `invalid-plan` | `not a valid plan store: <path>` |
| `invalid-plan` | `plan has no members` or `plan has no accounts` |
| `invalid-plan` | `S:<assumption-path>` (one line per missing field) |
| `invalid-plan` | `allocation sums to <N>% in year <Y>` (one line per violation) |

### Validation

Before running the projection, the CLI validates:

1. **Directory exists** — if `<dir>` does not exist, exit 1:
   `error: system-error: No such file or directory (os error 2)`.
   If the directory exists but is not a valid plan store, exit 1:
   `error: invalid-plan: not a valid plan store: <path>`.
2. **No members or no accounts** — a plan with no members or no accounts is
   invalid; exit 1 with one of:
   `error: invalid-plan: plan has no members`
   `error: invalid-plan: plan has no accounts`
3. **Required assumptions present** — all assumption fields with no shipped
   default must be populated. Missing fields cause exit 1; one line per missing
   field on stderr:
   `error: invalid-plan: S:assumptions / inflation / cpi`
4. **Allocations sum to 100%** — all allocation streams must sum to exactly 100%
   for every active year. Any year in violation causes exit 1; one line per
   violation on stderr:
   `error: invalid-plan: allocation sums to <N>% in year <Y>`

Validation failures are reported to stderr; stdout receives nothing.

### Denomination

All monetary values are output in YNV (year nominal value) — each year's values
are denominated in that year's nominal dollars. No denomination conversion is
applied. No denomination toggle is provided.

### Data Shape

All output formats share the same orientation: **columns are years, rows are
streams**. The first column is a row label; subsequent columns are one per
projection year in ascending order.

Row labels use user-defined names from the plan: `Member.given_name` for members,
`Account.label` for accounts.

Rows emitted, in order:

| Row label | Content |
|-----------|---------|
| `age.<given_name>` | Age of the member in that year (one row per member) |
| `bal.<label>` | End-of-year account balance (one row per account) |

### CSV Output (default)

RFC 4180 CSV written to stdout. First row is a header: `stream,<year>,<year>,...`.
One data row per stream. No quoting unless a field contains a comma. Monetary
values are bare numbers rounded to two decimal places — no currency symbols.
Age values are integers.

### Table Output (`--table`)

The same row/column orientation as CSV, formatted as a standard ASCII table.
Layout rules:

- Each column is fixed-width: the width of the widest value in that column
  (including the header), with a minimum of 4 characters for value columns
  (the width of a 4-digit year header).
- Columns are separated by two spaces.
- The label column is left-aligned. All value columns are left-aligned.
- The header row is followed by a separator row of `-` characters, one `-`
  per character position across the full row width (including inter-column spacing).
- No border characters on the left or right edges.
- No trailing whitespace on any line.

Example (illustrative widths only):

```
stream               2025      2026      2027
-------------------------------------------
age.Alice            45        46        47
bal.401k             123456.78 130000.00 137000.00
```

Intended for manual inspection; not suitable for programmatic parsing.

### JSON Output (`--json`)

A single JSON object written to stdout:

```json
{
  "denomination": "ynv",
  "years": [2025, 2026, ...],
  "rows": [
    { "stream": "age.<given_name>", "values": [45, 46, ...] },
    { "stream": "bal.<label>", "values": [123456.78, 130000.00, ...] },
    ...
  ]
}
```

- `denomination` is always `"ynv"`. Each year's values are in that year's nominal dollars.
- `years` lists the projection years in ascending order.
- `rows` is an array of objects in the same row order as CSV. Each object has
  a `stream` label and a `values` array parallel to `years`.
- Monetary values are JSON numbers (not strings), rounded to two decimal places
  in output only; internal computation remains full-precision. No currency symbols.
- Age values are JSON integers.

---

## Generate Mode

`--generate-plan <dir>` creates a minimal plan directory at the given path. The
path must not already exist; all filesystem failures exit 1 with the OS error
message verbatim using the `system-error` type (e.g.,
`error: system-error: File exists (os error 17)`,
`error: system-error: Permission denied (os error 13)`,
`error: system-error: No such file or directory (os error 2)`).

The generated plan contains:

- One household member with `birth_year = current_year − 35`, `death_age = 90`,
  `retirement_age = 65`.
- One retirement lifecycle event for that member at `age = 65`.
- One traditional 401k account with a starting balance.
- One `Contribution401kEmployeeStream` with `start = MemberAge(M.id, age: 22)`
  and `terminates = OnEvent(retirement_event_id)`.
- One bank account with a starting balance.
- One plan-level `AllocationStream` with a single anchor point: placeholder
  percentages across large-cap, small-cap, international, and bonds summing to 100%.
- A full set of assumptions populated with placeholder values (CPI inflation rate,
  return rates for each asset class).

`anchor_year` is written as a top-level field in the generated plan and equals
the current calendar year at generation time. All monetary placeholder values
are YZV denominated in that `anchor_year`.

After writing the plan files, the CLI runs projection-mode validation on the
generated directory. If validation fails, the CLI exits 1 with the validation
error on stderr. A successful `--generate-plan` run guarantees that
`yarp-cli <dir>` exits 0 immediately afterward without user edits.

The generated files are in `JsonPlanStore` format — they are directly editable JSON.

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Any error (validation failure, conflicting flags, I/O error, plan load failure) |

Error messages go to stderr. Successful output goes to stdout. Nothing is
interleaved.
