# Implementation Principles

These principles govern all implementation work. They are guards against specific
antipatterns, not general advice.

---

## No Magic Numbers

Numeric literals embedded in logic are wrong. IRS limits, RMD divisors, tax rates,
bracket thresholds, and any other domain constant belong in the Properties object or
a named constants module — never inline in a calculation.

**Antipattern:** `balance * 0.035` or `if age >= 73` appearing in computation logic
without a named constant explaining what the value is and where it comes from.

---

## Branching Logic

The design principle in [design/principles.md](../design/principles.md) establishes
that behavior varying by type or strategy belongs in the object model, not in branches.
At the implementation level this means:

**Dispatch tables over `if/else if` chains.** When branching on a type, category, or
strategy value, replace the chain with a map from keys to callables. The
dispatch table is data; the logic lives in the callables.

```rust
// Wrong
if account_type == "roth" {
    // ...
} else if account_type == "traditional" {
    // ...
}

// Right
let handlers: HashMap<&str, fn(&Account)> = HashMap::from([
    ("roth", handle_roth as fn(&Account)),
    ("traditional", handle_traditional as fn(&Account)),
]);
handlers[account_type](account);
```

**No behavior-switching parameters.** Any parameter whose purpose is to select which
logic path to execute — boolean flags, enums used as switch cases, string mode
selectors, type discriminators — is a design smell. The function is multiple behaviors
pretending to be one. The test: if removing the parameter would require splitting the
function, the parameter is controlling behavior, not providing data.

The right resolution depends on context: if the variation is tied to an entity, it
belongs on the type (trait impl); if it is a pluggable strategy, it belongs in a
callable strategy object (trait object or closure).

```rust
// Wrong — boolean flag
fn compute_withdrawal(amount: Decimal, is_roth: bool) -> Decimal { /* ... */ }

// Wrong — enum as switch
fn compute_withdrawal(amount: Decimal, account_type: AccountType) -> Decimal { /* ... */ }

// Wrong — string mode selector
fn compute_withdrawal(amount: Decimal, strategy: &str) -> Decimal { /* ... */ }

// Right — variation tied to account type: trait impl
trait Withdrawable {
    fn compute_withdrawal(&self, amount: Decimal) -> Decimal;
}

struct RothAccount { /* fields */ }
impl Withdrawable for RothAccount {
    fn compute_withdrawal(&self, amount: Decimal) -> Decimal { /* ... */ }
}

struct TraditionalAccount { /* fields */ }
impl Withdrawable for TraditionalAccount {
    fn compute_withdrawal(&self, amount: Decimal) -> Decimal { /* ... */ }
}

// Right — variation is a pluggable strategy: trait object
trait WithdrawalStrategy {
    fn apply(&self, amount: Decimal) -> Decimal;
}
```

**Early returns over nested conditionals.** Guard clauses at the top of a function
(validate and return early) are preferred over wrapping the main logic in an `if`
block. This keeps the happy path unindented and the complexity flat.

```rust
// Wrong
fn process(x: Option<f64>) -> Option<f64> {
    if let Some(val) = x {
        if val > 0.0 {
            let result = compute(val);
            return Some(result);
        }
    }
    None
}

// Right
fn process(x: Option<f64>) -> Option<f64> {
    let val = x?;
    if val <= 0.0 {
        return None;
    }
    Some(compute(val))
}
```

---

## Boundary Conditions From Architecture, Not Conditional Logic

This is the implementation expression of the bedrock Stream-Based Computation
principle. At the code level, the antipattern is `if` guards that check whether
a stream is active at a given year inside a computation function.

**Antipattern:** `if year == projection_start_year` or `if year >= rmd_start_year`
guards inside a computation function. These are fragile (every new boundary needs
another conditional), untestable in isolation (the boundary logic is tangled with the
computation), and invisible (a missing guard produces a silent wrong answer, not an
error).

```rust
// Wrong — consumer checks which assets are active
fn compute_net_worth(year: u32, assets: &[Asset], liabilities: &[Liability]) -> Decimal {
    let mut total = Decimal::ZERO;
    for asset in assets {
        if year >= asset.purchase_year && !asset.is_depleted(year) {
            total += asset.value_at(year);
        }
    }
    for liability in liabilities {
        if year >= liability.start_year && year <= liability.end_year {
            total += liability.balance_at(year); // negative
        }
    }
    total
}

// Right — streams yield zero for inactive years; consumer just sums
fn compute_net_worth(year: u32, model: &Model) -> Decimal {
    let assets: Decimal = model.asset_streams().map(|s| s.value_at(year)).sum();
    let liabilities: Decimal = model.liability_streams().map(|s| s.balance_at(year)).sum();
    assets + liabilities
}
```

---

## Function and Method Argument Count

Keep argument lists short. A function that requires many parameters to do its job is
a signal that those parameters belong together as a structured object. Introduce an
argument aggregator — a struct — that groups related inputs and makes
the signature extensible without modification.

```rust
// Wrong — grows unboundedly as requirements change
fn project_asset(
    balance: Decimal, return_rate: Decimal, contribution: Decimal,
    inflation: Decimal, year: u32, phase: LifePhase, is_retired: bool,
) -> Decimal {
    // ...
}

// Right — aggregator groups related inputs; adding a field doesn't touch the signature
struct AssetProjectionInputs {
    balance: Decimal,
    return_rate: Decimal,
    contribution: Decimal,
    inflation: Decimal,
    year: u32,
    phase: LifePhase,
}

fn project_asset(inputs: &AssetProjectionInputs) -> Decimal { /* ... */ }
```

Before introducing a new aggregator, ask whether the parameters being grouped are
better served as properties on the Properties singleton. If multiple components need
the same values, they belong in Properties — not in an aggregator that must be
constructed and threaded through each call site.

If the values are genuinely local to one function's context, prefer building the
aggregator hierarchically: extend or compose an existing aggregator rather than
creating a new one. A codebase with hundreds of one-off aggregator classes is not
meaningfully better than one with long argument lists — the complexity is the same,
just differently shaped.

**Antipattern:** creating a new aggregator class for every function that has more than
two arguments. Aggregators should reflect stable, reusable groupings of related data —
not be a mechanical response to argument count.

The aggregator must not become a workaround for the no-behavior-switching-parameters
rule. Bundling a behavior-switching parameter into an aggregator object just hides the
violation — the selector still exists and the function still branches on it. Behavior
variation belongs in the object model, not in aggregated parameters.

---

## Cyclomatic Complexity

Keep cyclomatic complexity per function low — a target of 3, a hard limit of 6. A
function that exceeds the limit must be decomposed before the code is considered
complete.

High complexity is a diagnostic signal, not just a style issue. It means the function
is difficult to test exhaustively (each path needs a test case), difficult to reason
about, and likely encoding logic that belongs in the object model. A complexity spike
almost always indicates that a design abstraction is missing — see the Functors and
Active Objects and OOP principles in [design/principles.md](../design/principles.md).

Single-purpose helper functions are an approved decomposition strategy. A function
that exists solely to give a name to one clause of a larger operation is not overhead
— it is the correct tool for keeping the calling function within complexity limits and
making intent explicit.

**Antipattern:** a single function handling multiple account types, tax regimes, or
withdrawal strategies through nested conditionals. Each variation should be a distinct
object or function, not a branch.

---

## Error Handling

Errors signal unexpected failure — they are not a control flow mechanism. Do not
use error results to implement expected branching (e.g., returning an error to signal
"no result found" when `Option` would suffice). Do not silently discard errors.
Do not use catch-all error handlers that erase the original error type.

Financial calculations must fail loudly. A silently discarded error in a projection
pipeline can produce a plausible-looking but numerically wrong result — which is worse
than an obvious crash.

```rust
// Wrong — catch-all discards everything
fn try_compute_rmd(balance: Decimal, divisor: Decimal) -> Decimal {
    match compute_rmd(balance, divisor) {
        Ok(val) => val,
        Err(_) => Decimal::ZERO, // silently swallows every error
    }
}

// Wrong — error as control flow
fn find_bracket(income: Decimal, brackets: &[Bracket]) -> Result<&Bracket, String> {
    for bracket in brackets {
        if bracket.contains(income) {
            return Ok(bracket);
        }
    }
    Err("not found".into()) // use Option, not Result, for expected absence
}

// Right — explicit error type, propagate or convert with context
fn compute_rmd(balance: Decimal, divisor: Decimal) -> Result<Decimal, RmdError> {
    if divisor.is_zero() {
        return Err(RmdError::ZeroDivisor { age });
    }
    Ok(balance / divisor)
}
```

---

## Implicit None Returns

Every function must either return a typed value or return an error. A function that
can silently produce a meaningless default — by falling through a conditional without
an explicit return — forces every call site to guard against unexpected values. When a
caller omits the guard, a silent zero or empty value propagates. In a financial
calculation, this is a class of silent numeric error.

```rust
// Wrong — returns Decimal::ZERO implicitly if balance <= 0, caller may not notice
fn compute_rmd(balance: Decimal, divisor: Decimal) -> Decimal {
    if balance > Decimal::ZERO {
        return balance / divisor;
    }
    // implicit: falls through to... what? Rust requires all paths return,
    // but the principle is: be explicit about every exit path.
    Decimal::ZERO
}

// Right — explicit about all exit paths
fn compute_rmd(balance: Decimal, divisor: Decimal) -> Decimal {
    if balance <= Decimal::ZERO {
        return Decimal::ZERO;
    }
    balance / divisor
}
```

---

## Mixing Abstraction Levels

A function must not mix high-level orchestration with low-level detail in the same
body. If a function coordinates several steps AND implements one of those steps inline,
the result is hard to read, hard to test, and hard to modify. Extract the inline detail
into a single-purpose helper.

```rust
// Wrong — orchestration and detail mixed
fn project_year(state: &State) -> Decimal {
    // high-level step 1
    let mut contributions = state.salary * state.contribution_rate;
    // inline detail
    if state.age >= 50 {
        contributions += CATCH_UP_LIMIT;
    }
    if contributions > IRS_LIMIT {
        contributions = IRS_LIMIT;
    }
    // high-level step 2
    let growth = (state.balance + contributions) * state.return_rate;
    state.balance + contributions + growth
}

// Right — detail extracted; orchestrator reads as a sequence of named steps
fn project_year(state: &State) -> Decimal {
    let contributions = compute_contributions(state);
    let growth = compute_growth(state.balance, contributions, state.return_rate);
    state.balance + contributions + growth
}
```

---

## Stringly-Typed Code

Use enums for categorical values — never raw strings. Strings have
no type safety: a typo is a silent runtime bug. An enum is checked
by the compiler and navigable by tooling.

```rust
// Wrong — string literal has no type safety
if account.account_type == "roth" {
    // ...
}
if withdrawal_strategy == "proportional" {
    // ...
}

// Right — enum variants are checked at compile time; match is exhaustive
enum AccountType {
    Roth,
    Traditional,
}

enum WithdrawalStrategy {
    Proportional,
    Sequential,
}

match account.account_type {
    AccountType::Roth => { /* ... */ }
    AccountType::Traditional => { /* ... */ }
}
```

---

## Mutable Default Arguments

Rust does not have mutable default arguments (a Python-specific pitfall), but the
underlying principle still applies: function signatures must not share hidden mutable
state across calls. When an optional collection is needed, use `Option` and
construct a new instance inside the function body.

```rust
// Right — caller can pass an existing Vec or let the function create one
fn add_contribution(amount: Decimal, contributions: Option<Vec<Decimal>>) -> Vec<Decimal> {
    let mut contributions = contributions.unwrap_or_default();
    contributions.push(amount);
    contributions
}
```
