# Testing Principles

These principles govern all test code. They are guards against specific
antipatterns, not general advice.

---

## Test Behavior, Not Implementation

Tests assert the behavior specified in requirements and design — inputs, outputs,
invariants — not internal implementation details. A test that breaks when you
refactor without changing behavior is testing the wrong thing.

---

## Cover Boundaries Explicitly

The general case is rarely where failures hide. Boundary years (first projection year,
retirement year, RMD start, death year), empty inputs, and maximum values must be
covered explicitly — not assumed to fall out of the general case.

---

## Regression Against Known Outputs

The engine must be verifiable against pinned fixtures. Regression tests use a
reference household configuration with known expected outputs, not just "it runs
without error."
Tolerances for floating-point comparison must be specified explicitly.

---

## Invariants Are Assertions, Not Documentation

If a constraint must hold (allocations sum to 100%, RMDs are non-negative, net worth
equals assets minus liabilities), it must be asserted in a test. Stating an invariant
in a comment or document is not a substitute for asserting it in code.
