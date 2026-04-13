# CLI Design

> **Design-for-test scope.** This CLI is a headless test harness, not a user
> product. It exists to make the engine scriptable from CI pipelines and
> integration tests. Ease-of-use is explicitly out of scope. The interface is
> minimal by design.

---

## Architecture Position

The CLI is a standalone Rust binary crate (`yarp-cli`) that links the engine as
a library dependency and calls it directly. It does not go through the
Controller, the Model facade, or the Tauri IPC layer. It reads plan data from the
filesystem using `JsonPlanStore` — the same format the production backend uses —
and writes output to stdout. No UI state exists.

```
yarp-cli  →  JsonPlanStore  →  Engine  →  stdout
```

---

## Invocation Modes

The CLI has exactly two mutually exclusive modes:

| Mode | When | Description |
|------|------|-------------|
| **Projection** | a positional `<dir>` argument is given (default) | Load plan, run projection, write table |
| **Generate** | `--generate-plan <dir>` is given | Write a minimal runnable plan directory |

Supplying both a positional directory and `--generate-plan` in the same
invocation is an error (exit 1, message to stderr). Supplying both `--json`
and `--table` in the same invocation is also an error (exit 1, message to stderr).

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

### Validation

Before running the projection, the CLI validates:

1. **Directory exists** — if `<dir>` does not exist, exit 1 with message to
   stderr: `error: directory not found: <path>`. If the directory exists but is
   not a valid plan store, exit 1 with: `error: not a valid plan store: <path>`.
2. **No members or no accounts** — a plan with no members or no accounts is
   invalid; exit 1 with a descriptive message to stderr.
3. **Required assumptions present** — all assumption fields with no shipped
   default must be populated. Missing fields cause exit 1; one line per missing
   field on stderr using `S:` path syntax (e.g.,
   `error: missing required assumption: S:assumptions / inflation / cpi`).
4. **Allocations sum to 100%** — all allocation streams must sum to exactly 100%
   for every active year. Any year in violation causes exit 1; one line per
   violation on stderr: `error: allocation sums to <N>% in year <Y>`.

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
values are bare numbers — no currency symbols.

### Table Output (`--table`)

The same row/column orientation as CSV, formatted as an aligned text table.
Columns are space-padded to a fixed width determined by the widest value in
each column. The header row is separated from data rows by a `-` rule.
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
path must not already exist; if it does, the CLI exits 1 with an error to stderr.

The generated plan contains:

- One household member with `birth_year = current_year − 35`, `death_age = 90`,
  `retirement_age = 65`.
- One traditional 401k account with a starting balance.
- One bank account with a starting balance.
- A full set of assumptions populated with placeholder values (inflation rates,
  return rates, allocations summing to 100%).
- One retirement lifecycle event.

All monetary placeholder values are YZV denominated in `anchor_year`, where
`anchor_year` equals the current calendar year at generation time.

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
