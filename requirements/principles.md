# Requirements Principles

These principles govern all requirements artifacts. They are guards against specific
antipatterns, not general advice.

---

## Solutioning Is Not a Requirement

A requirement states what the system must do or what the user must experience. It
does not prescribe how.

**Antipattern:** "The system shall display a dropdown menu for account selection."
That is a UX design decision. The requirement is: "The user must be able to select
from their available accounts."

**Antipattern:** "RMD amounts shall be stored in the `rmd_amount` field of the
timeline JSON." That is a design decision. The requirement is: "The system must
track RMD obligations per year."

**The test:** could this requirement be satisfied by two fundamentally different
designs? If not, it's specifying a solution, not a problem.

---

## Testable Without Implementation

Every requirement must be verifiable by someone who cannot see the code. If a
requirement can only be confirmed by inspecting internals, it is either a design
constraint or an implementation detail mislabeled as a requirement.

**Antipattern:** "The projection engine shall use a single-pass forward sweep."
That describes an algorithm (design). The requirement is: "Projected values for
year N must depend only on inputs and values from years ≤ N."

**Antipattern:** "Account balances shall use Decimal types." That is an
implementation constraint. The requirement is: "All monetary calculations must
be accurate to the cent over 40-year projections."

---

## One Behavior Per Requirement

A requirement that bundles multiple behaviors cannot be partially implemented,
partially tested, or partially satisfied. Split compound requirements into
independently addressable statements.

**Antipattern:** "The system must compute RMDs, apply them as withdrawals, and
display the results in the timeline view." That is three requirements: computation,
application, and display.

---

## Domain Language, Not Tech Language

Requirements use the vocabulary of the problem domain — retirement planning, tax
law, financial projections — not the vocabulary of software engineering. A domain
expert should be able to read and validate every requirement without knowing
what language the system is written in.

**Antipattern:** "The API shall return a JSON payload containing projected balances."
The requirement is: "The user must be able to retrieve projected balances for any
scenario."

**Antipattern:** "The system shall emit events when assumptions change." Events are
a design mechanism. The requirement is: "Projections must update when underlying
assumptions are modified."

---

## Acceptance Criteria Are Mandatory

A requirement without acceptance criteria is a wish. Every requirement must include
conditions under which it can be declared satisfied. These conditions are stated in
terms of observable behavior, not internal state.

**Antipattern:** "The system should handle edge cases for Social Security." What edge
cases? Handle how? The requirement must enumerate the specific scenarios and expected
outcomes.

---

## No Speculative Requirements

Requirements describe what the system must do, not what it might someday do. A
requirement that exists because "it might be useful" consumes design and implementation
effort without a clear stakeholder need.

Future capabilities belong in a roadmap or planning document, not in the requirements
tree. If forward-looking context is needed for architectural decisions, it is recorded
as a design consideration, not as a requirement.

**Antipattern:** "The system should support Monte Carlo simulation for stress testing."
If no one has asked for this and no current feature depends on it, it is not a
requirement — it is speculation that will drive unnecessary design complexity.
