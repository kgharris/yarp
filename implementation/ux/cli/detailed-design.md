# CLI Detailed Design

## Overview

This document is the detailed design for the CLI binary (`yarp-cli`). It
captures implementation-level decisions — types, algorithms, formatting
rules, and their relationships — at a level below
[design/ux/cli.md](../../../design/ux/cli.md) but above source code.

The CLI is the View layer: given a plan directory, it loads the plan via
the Model facade, runs the projection, and writes formatted output to
stdout. It performs zero financial computation. For workspace layout, crate
structure, and dependencies, see the
[project detailed design](../../detailed-design.md).

All CLI logic lives in `crates/yarp-cli/src/main.rs`.

Authoritative design specifications:

- [cli.md](../../../design/ux/cli.md) — modes, flags, output formats, row
  ordering, data shape
- [ux principles](../../../design/ux/principles.md) — MVC constraints
- [api.md](../../../design/engine/api.md) — Model facade, Projection output
- [error-handling.md](../../../design/error-handling.md) — error format,
  caller contract
- [data-model.md](../../../design/data-model.md) — stream primitives,
  aggregate tree structure

### Data Flow

Projection mode:

```
parse args → resolve Invocation::Project
  → Model::load(dir) → PlanContext
  → Model::get_projection(&plan_ctx) → Projection
  → emit_projection(plan_ctx, projection, formatter) → stdout
```

Generate mode:

```
parse args → resolve Invocation::Generate
  → Model::generate(dir, params) → ()
  → println!("Plan created in {dir}")
```

Errors at any stage propagate as `CliError` to `main()`, which writes the
formatted error to stderr and exits 1.

---

## Public API Surface

The CLI imports these types from `yarp-core`:

**Facade:**
- `Model` — `load()`, `generate()`, `get_projection()` (associated functions)
- `GeneratePlanParams` — `{ anchor_year: i32 }`

**Output types:**
- `PlanContext` — `{ graph: PlanGraph, ... }`
- `PlanGraph` — entity collections; `owner_label(&OwnedBy) -> &str`
  accessor
- `Projection` — `{ years, root_id, streams, points }`
- `Stream` — `{ id, kind, label, owner, procedure, inputs, ... }`
- `StreamPoint` — `{ year, entries }`
- `OwnedBy` — all variants
- `StreamKind` — for `MemberLifecycleStream` detection
- `PointDenomination` — for `Idv` detection (inactive year rendering)
- `Decimal` (re-exported from `rust_decimal`)

**Error type:**
- `ModelError` — `Display` impl produces formatted error strings per
  [error-handling.md](../../../design/error-handling.md)

The CLI does **not** access engine internals (`eval`,
`resolved_start`, `resolved_end`, procedures), persistence internals
(`PlanStore`, `JsonPlanStore`, `PlanStoreError`), or stream construction APIs.

---

## Argument Parsing

```rust
#[derive(clap::Parser)]
#[command(name = "yarp-cli")]
struct Args {
    /// Path to plan directory (projection mode)
    #[arg(conflicts_with = "generate_plan")]
    dir: Option<PathBuf>,

    /// Emit JSON to stdout instead of the default CSV
    #[arg(long, group = "format")]
    json: bool,

    /// Emit an aligned text table to stdout instead of the default CSV
    #[arg(long, group = "format")]
    table: bool,

    /// Write an initial plan directory to this path
    #[arg(long, value_name = "DIR")]
    generate_plan: Option<PathBuf>,
}
```

- `dir` is `Option<PathBuf>` because no-args prints usage.
- `conflicts_with` enforces mutual exclusivity of dir and generate at parse
  time.
- `group = "format"` enforces `--json`/`--table` mutual exclusivity.
- No `--csv` flag. CSV is the default.

### OutputFormat

```rust
enum OutputFormat { Csv, Table, Json }
```

### Invocation

```rust
enum Invocation {
    Usage,
    Generate { dir: PathBuf },
    Project { dir: PathBuf, format: OutputFormat },
}

fn resolve_args(args: Args) -> Result<Invocation, String>
```

Validation:
- Both `dir` and `generate_plan` are `None` → `Ok(Usage)`
- `generate_plan` is `Some(dir)` → `Ok(Generate { dir })`
- `dir` is `Some(dir)` → `Ok(Project { dir, format })`
- Both present → `Err(...)` (unreachable due to `conflicts_with`, kept for
  defense-in-depth)

`resolve_args` is pure — no `process::exit`, no side effects.

> **Note:** The CLI Integration section in
> [implementation/detailed-design.md](../../detailed-design.md) contains a
> stale `Args` sketch that diverges from the authoritative
> [design/ux/cli.md](../../../design/ux/cli.md). This spec is
> self-contained and follows the design spec. The project detailed design
> needs to be updated to match.

---

## Error Handling

```rust
enum CliError {
    Model(ModelError),
    Io(std::io::Error),
}
```

`From<ModelError>` and `From<std::io::Error>` impls enable `?` propagation.
`Display` delegates to `ModelError::Display` for `Model` variants, and
formats I/O errors as `error: system-error: <msg>`.

Single exit point in `main()`:

```rust
fn main() {
    let args = Args::parse();
    match resolve_args(args) {
        Ok(Invocation::Usage) => {
            Args::command().print_help().unwrap();
        }
        Ok(Invocation::Generate { dir }) => {
            if let Err(e) = run_generate(&dir) {
                eprintln!("{e}");
                std::process::exit(1);
            }
        }
        Ok(Invocation::Project { dir, format }) => {
            if let Err(e) = run_project(&dir, format) {
                eprintln!("{e}");
                std::process::exit(1);
            }
        }
        Err(msg) => {
            eprintln!("{msg}");
            std::process::exit(1);
        }
    }
}
```

Exit codes: 0 success, 1 any error. Clap exits with code 2 for parse
errors (e.g., `--json --table`); this is an accepted deviation from the
design spec's blanket exit-1 rule, as exit 2 for usage errors is standard
Unix convention.

---

## Projection Mode

```rust
fn run_project(dir: &Path, format: OutputFormat) -> Result<(), CliError> {
    let plan_ctx = Model::load(dir)?;
    let projection = Model::get_projection(&plan_ctx)?;
    let stdout = std::io::stdout();
    let writer = stdout.lock();
    let mut formatter: Box<dyn Formatter> = match format {
        OutputFormat::Csv => Box::new(CsvFormatter::new(writer)),
        OutputFormat::Table => Box::new(TableFormatter::new(writer)),
        OutputFormat::Json => Box::new(JsonFormatter::new(writer)),
    };
    emit_projection(&plan_ctx, &projection, &mut *formatter)?;
    Ok(())
}
```

---

## Generate Mode

```rust
fn run_generate(dir: &Path) -> Result<(), CliError> {
    let params = GeneratePlanParams {
        anchor_year: chrono::Local::now().year(),
    };
    Model::generate(dir, params)?;
    println!("Plan created in {}", dir.display());
    Ok(())
}
```

---

## Formatter Trait

The formatter is a functor that the tree walk drives. The walk calls
`header()` once, `row()` for each output row, and `finish()` at the end.

```rust
trait Formatter {
    fn header(&mut self, years: &[i32]) -> Result<(), CliError>;
    fn row(&mut self, label: &str, values: &[Option<CellValue>]) -> Result<(), CliError>;
    fn finish(&mut self) -> Result<(), CliError>;
}

enum CellValue {
    Age(i32),
    Money(Decimal),
}
```

`CellValue` is a lightweight enum used only in the functor call — not a
stored intermediate type. `Age` is determined by `StreamKind::MemberLifecycleStream`.
`Money` is used for all balance and aggregate streams.

Formatting rules for values:
- `CellValue::Age(v)` → integer string (no decimal point)
- `CellValue::Money(v)` → `v.round_dp(2).to_string()` (bare number, 2
  decimal places, no currency symbol)
- `None` → inactive year (format depends on output mode)

---

## Tree Walk

```rust
fn emit_projection(
    plan_ctx: &PlanContext,
    projection: &Projection,
    formatter: &mut impl Formatter,
) -> Result<(), CliError>
```

Uses `Projection`'s existing indexed collections for stream lookup — no
redundant index construction.

**Algorithm:**

1. Call `formatter.header(&projection.years)`.

2. **Lifecycle rows first.** Iterate `plan_ctx.graph.members` in insertion
   order. For each member, find the projection stream with
   `kind == MemberLifecycleStream` and `owner == Member(member_id)`.
   Label: `plan_graph.owner_label(&stream.owner)` (returns `given_name`).
   Values: for each year, read `entries["age"]` from the stream's points;
   `Some(CellValue::Age(...))` if present, `None` if absent. Call
   `formatter.row()`.

3. **Aggregate tree walk.** Use `projection.root_id` as the entry point.
   Call `walk_aggregate(root_id, formatter)`.

4. Call `formatter.finish()`.

### walk_aggregate

```
walk_aggregate(stream_id, formatter):
    stream = projection.get(stream_id)
    for (_, input_id) in stream.inputs:        // IndexMap preserves insertion order
        input = projection.get(input_id)
        if input found and procedure == Additive:
            walk_aggregate(input_id, formatter)
        else if input found:
            // leaf balance stream
            label = plan_graph.owner_label(&input.owner)
            values = extract values for each year
            formatter.row(label, &values)
        else:
            skip   // non-balance inputs (rates, contributions) not in projection
    // after all children, emit the aggregate itself
    label = stream.label
    values = extract values for each year
    formatter.row(label, &values)
```

### Row Labels

No prefixes. The tree structure provides context.

- **Leaf rows:** `plan_graph.owner_label(&stream.owner)` — the entity's
  display name (`"Alice 401k"`, `"Home"`, `"Mortgage"`)
- **Lifecycle rows:** `plan_graph.owner_label(&stream.owner)` — the
  member's given name (`"Alice"`, `"Bob"`)
- **Aggregate rows:** `stream.label` — the stream's own label
  (`"retirement"`, `"net-worth"`, `"assets"`)

### Inactive Year Handling

The projection contains a point for every stream for every year in the
timeline. The CLI distinguishes inactive years by checking the point's
denomination: `Idv` means the stream is outside its active range.
Points with `Idv` denomination render as `None` (empty cell in CSV,
blank in table, `null` in JSON). Points with `Ynv` denomination render
their value — including zero for depleted-but-active streams.

---

## CSV Formatter

```rust
struct CsvFormatter<W: Write> { writer: W }
```

RFC 4180 compliant. Uses `\r\n` line endings (not `writeln!` which
produces `\n` on Unix).

- `header()`: write `stream,<year>,<year>,...\r\n`
- `row()`: write `<label>,<val>,<val>,...\r\n`
  - `None` → empty field (nothing between commas)
  - `CellValue::Age(v)` → integer string
  - `CellValue::Money(v)` → `v.round_dp(2).to_string()`
- `finish()`: no-op

No quoting needed — labels are validated comma-free at plan load time.

---

## Table Formatter

```rust
struct TableFormatter<W: Write> {
    writer: W,
    years: Vec<i32>,
    labels: Vec<String>,
    values: Vec<Vec<Option<CellValue>>>,
}
```

Buffers all rows internally (needs column widths before writing).

- `header()`: store years.
- `row()`: buffer label and values.
- `finish()`: compute widths, write everything:

  1. **Column widths.** Label column: max of `"stream".len()` and all
     label lengths. Value columns: max of year header width (4) and widest
     formatted value. Minimum 4 characters per value column.

  2. **Header.** Label left-aligned, years right-aligned, columns
     separated by two spaces. `trim_end()` before writing. `\n` terminator.

  3. **Separator.** `-` characters spanning full row width. `\n` terminator.

  4. **Data rows.** Label left-aligned, values right-aligned.
     `None` → spaces to fill column width. `trim_end()` before writing.
     `\n` terminator.

Table output uses `\n` line endings (not `\r\n`). Intended for manual
inspection, not programmatic parsing.

---

## JSON Formatter

```rust
struct JsonFormatter<W: Write> {
    writer: W,
    years: Vec<i32>,
    rows: Vec<JsonRow>,
}
```

Buffers all rows (JSON requires the complete object before writing).

- `header()`: store years.
- `row()`: convert values to `Vec<JsonValue>`, buffer as `JsonRow`.
- `finish()`: serialize `JsonOutput` with `serde_json::to_writer_pretty`.

Serialization types:

```rust
#[derive(serde::Serialize)]
struct JsonOutput {
    denomination: &'static str,  // always "ynv"
    years: Vec<i32>,
    rows: Vec<JsonRow>,
}

#[derive(serde::Serialize)]
struct JsonRow {
    stream: String,
    values: Vec<JsonValue>,
}

#[derive(serde::Serialize)]
#[serde(untagged)]
enum JsonValue {
    Integer(i32),
    Number(f64),
    Null,
}
```

Conversion:
- `None` → `JsonValue::Null`
- `Some(CellValue::Age(v))` → `JsonValue::Integer(v)`
- `Some(CellValue::Money(v))` → `JsonValue::Number(v.round_dp(2).to_f64().unwrap())`
  Panics if the Decimal cannot be represented as f64. At 2dp rounding, values
  up to ~10^13 are safe — well beyond retirement planning range.

---

## Cargo Dependencies

```toml
[dependencies]
yarp-core = { path = "../yarp-core" }
clap = { workspace = true, features = ["derive"] }
serde = { workspace = true, features = ["derive"] }
serde_json = { workspace = true }
chrono = "0"
```

---

## Test Cases

**Unit tests** (inline `#[cfg(test)] mod tests` in `main.rs`):

- `resolve_args`: no args → Usage, dir only → Project/Csv, dir + json →
  Project/Json, dir + table → Project/Table, generate-plan → Generate
- CSV formatter: header format, monetary rounding (2dp), age as integer,
  empty cells (`,,`), CRLF line endings
- Table formatter: column alignment, separator row width, min 4-char value
  columns, trailing whitespace trimmed, blank cells padded
- JSON formatter: schema structure (`denomination`, `years`, `rows`),
  `null` for inactive, age as integer, monetary as number rounded to 2dp

Formatter tests call `header()`, `row()`, `finish()` with hand-constructed
`CellValue` slices, using `Vec<u8>` as the `Write` target for byte-exact
comparison. No engine types needed.

**Integration tests** (`crates/yarp-cli/tests/cli_tests.rs`):

- Golden-file tests: generate a plan, project as CSV/JSON/table, compare
  against reference output
- Error tests: nonexistent directory, malformed plan, conflicting flags
- Generate roundtrip: `--generate-plan <tmpdir>` then `<tmpdir>` produces
  valid output
