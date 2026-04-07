# Engine — Implementation Principles

Implements the design in [/design/engine](../../design/engine/).

---

## Pure Functions

Engine modules are stateless. Given the same inputs, a module must produce the same
outputs. No module mutates shared state, reads from global variables (other than the
Properties singleton), or produces side effects.

This is the implementation-level expression of the design constraint that engine modules
are pure computation. It makes modules independently testable and the pipeline
deterministic.

**Antipattern:** a module that reads from a file, maintains cached internal state between
calls, or behaves differently depending on call order.

## Numeric Precision

All monetary values and rates are represented using a decimal type, not floating point.
Floating-point arithmetic introduces rounding errors that compound across 40-year
projections and produce results that appear correct but are wrong.

This applies to balances, rates, tax amounts, contributions, withdrawals, and any
intermediate value that feeds into a monetary calculation. Floating-point types are
acceptable only for non-monetary computations where precision loss is inconsequential
(e.g. age-based interpolation weights).
