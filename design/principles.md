# Design Principles

These principles govern all design artifacts. They are guards against specific
antipatterns, not general advice.

---

## Data-Driven Business Rules

Business rules that are subject to external change — tax brackets, IRS contribution
limits, RMD divisor tables, Social Security parameters, IRMAA tiers, ACA thresholds,
estate tax exclusions — must be represented as data, not as logic.

Algorithms operate on rule data; they do not embed rule values. When a bracket
threshold changes, a table entry changes. No algorithm changes. When the IRS adjusts
contribution limits, a configuration value changes. No code changes.

This applies at every layer of the design:
- The **DB layer** owns the storage and versioning of rule data
- The **Engine** receives rule data as input and applies it; it does not contain it
- The **Controller** resolves which rule data applies to a given tax year before
  passing it to the Engine

**Antipattern:** an algorithm that contains a literal like `0.22` for a tax rate, `73`
for RMD start age, or `$7,000` for an IRA limit. Any value that an IRS publication
could change belongs in rule data, not in an algorithm.

**The test:** if a tax law changes, should any algorithm change? No — only data
should change. If the answer is yes, the rule is embedded in the wrong place.

---

## Do Not Replicate Functionality

Common logic belongs in shared modules, classes, or functions — imported wherever
needed, not re-implemented per subsystem. If two places do the same thing, one of them
is wrong.

It is the design spec's responsibility to identify reusable components and name them.
Implementation provides exactly one canonical realization of each.

**Antipattern:** duplicating a calculation, parsing routine, or validation rule across
two modules because it was slightly more convenient than finding the shared location.

---

## Separation of Concerns Across the Entire Architecture

Subsystems interact through well-defined APIs. Components within a subsystem do the
same. No component reaches into another's internals. This applies at every scale:
subsystem boundaries, module boundaries, and class boundaries.

If a change to one module requires editing another module's internals, the boundary is
wrong. Fix the boundary, not the symptom.

---

## OOP and Polymorphism — Used Where It Earns Its Place

Introduce classes where they model a real entity or encapsulate behavior that varies by
type. Polymorphism is the correct tool when the same operation must behave differently
depending on context — account type, withdrawal strategy, tax regime, etc.

Do not replicate conditional logic across call sites when a class hierarchy resolves it
cleanly. But do not introduce classes for their own sake. The test: does this class
eliminate duplication or clarify a boundary? If not, it is overhead.

This principle is the structural expression of "do not replicate functionality."

---

## Properties as a Singleton for Cross-Cutting Configuration

Preferences, tunable parameters, and decision points that multiple subsystems need are
collected into a Properties object. This object is a singleton — initialized once,
accessible to any subsystem without threading it through every call signature.

This prevents the antipattern of passing configuration through long chains of function
arguments, and ensures there is one authoritative source for any tunable value.

---

## Functors and Active Objects Over Branching Logic

Conditional branching (`if/then/else`, `match/case`) is a code-rot vector. When
behavior varies by type or strategy, model the variation as a callable object (functor)
or active object rather than as a branch.

A function that dispatches to different logic based on an account type should be
replaced by a polymorphic object where each type carries its own behavior. Branching at
call sites that could be resolved by the object model is a signal that the object model
is incomplete.

---

## Fail Fast at Boundaries

Validate inputs at system entry points — API requests, file reads, external data — and
raise a typed error immediately. Do not let bad data propagate silently through the
computation pipeline.

Silent propagation is especially dangerous here: the engine may produce a
plausible-looking result that is numerically wrong. A corrupt or out-of-range input
should produce an immediate, unambiguous error, not a subtly incorrect projection.

---

## Design for Testability

Components must be designed so they can be exercised in isolation. A module whose
interface requires a full running system to instantiate has the wrong boundary.

Testability is a design constraint, not a testing phase. It is determined by how
interfaces and boundaries are drawn — not by how tests are later written. If a
component cannot be tested without standing up adjacent subsystems, the design
needs to be fixed, not the tests.

---

## Technology Notes — Rust Implementation

These notes map the OOP-flavored vocabulary in this document to Rust idioms.
The principles are unchanged; only the realization differs.

### Polymorphism and Abstract Interfaces
"Class hierarchy" and "abstract base class" map to **traits** in Rust.
A trait defines the interface; each concrete type implements it independently.
Default trait methods provide shared implementation — the equivalent of
non-abstract base class methods.

Runtime polymorphism (passing a derived type where a base is expected) uses
`&dyn Trait` or `Box<dyn Trait>`. Example: a `Vec<Box<dyn Account>>` holds
a mixed collection of `TraditionalIra`, `RothIra`, `Hsa`, etc., with
polymorphic dispatch on all trait methods.

Static dispatch (when the concrete type is known at compile time) uses
generics: `fn process<A: Account>(acct: &A)`. Prefer this over `dyn` where
possible — it has no runtime cost.

### Composition Over Inheritance
Rust has no inheritance. Shared fields are composed via embedding:
```rust
struct AccountBase { balance: f64, owner_id: String }
struct TraditionalIra { base: AccountBase }
```
Delegation to `self.base` is explicit. This is idiomatic and aligns with
the composition principle already stated above.

### Functors and Active Objects
"Callable objects" and "functors" map to structs that implement the `Fn`,
`FnMut`, or `FnOnce` traits, or more commonly in this domain, structs that
implement a domain trait with a single `execute` or `apply` method.
Enums with per-variant behavior (via trait dispatch) are the idiomatic
replacement for strategy pattern class hierarchies.

### Properties Singleton
Rust has no natural singleton. The idiomatic equivalent for a read-only
configuration object passed across subsystems is `Arc<Config>` — a
reference-counted shared pointer. For the engine pipeline, which is
sequential and single-threaded, passing config explicitly as a parameter
is simpler and preferred. The singleton is a conceptual pattern here;
do not implement it as mutable global state.

### Enums for Variants
Where Python or C++ would use a class hierarchy for a closed set of variants
(account types, filing status, withdrawal strategy, income model), Rust enums
with data are idiomatic and preferred. They are exhaustively checked by the
compiler, eliminating missing-case bugs.
