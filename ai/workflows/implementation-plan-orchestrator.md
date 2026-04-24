# Implementation Plan Orchestrator

## Purpose

Produces per-file implementation plans from detailed design documents. The
detailed design (`implementation/<component>/detailed-design.md`) captures
implementation-level decisions organized by topic. This orchestrator transforms
that topical content into per-file specifications that the
[implementation orchestrator](implementation-orchestrator.md) can dispatch to
code-writing agents.

This is a mechanical transformation, not a design process. All decisions have
already been made in the detailed design. The agents performing this work do
not need principles, design documents, or domain context — the detailed design
is their sole input.

## Usage

Provide:
- **component**: which subsystem to plan (`engine`, `db`, `controller`, `ux`)
- **scope** (optional): limit to a subset of files

If component is missing, prompt before proceeding.

**Inputs:**
- `implementation/detailed-design.md` — project-wide detailed design
- `implementation/<component>/detailed-design.md` — component detailed design

**Output:**
- `implementation/<component>/plan.md` — per-file implementation plan, ready
  for consumption by the [implementation orchestrator](implementation-orchestrator.md)

## Plan Format

The output plan follows the format expected by the
[implementation orchestrator](implementation-orchestrator.md). It is organized
as a sequence of **milestones**, each containing a set of **file entries**.
Within a milestone, file entries may declare dependencies on other files in
the same milestone (serial) or be independent (parallel).

```markdown
## Milestone M<N>: <title>

### `<file path>`

**Realizes:** <detailed design section(s) this file implements>
**Purpose:** <what this file does>
**Depends on:** <other file paths in this milestone, or "none">

**Specification:**
- Types, traits, functions to define (with signatures from detailed design)
- Behavioral requirements
- Constraints

**Tests (written first):**
- Unit tests to include in the inline `#[cfg(test)] mod tests` block
- Each test: name, inputs, expected output
- Tests that must pass before implementation is considered complete

**Acceptance criteria:**
- <observable conditions for this file being done>
```

## How It Works

The orchestrator reads the detailed design documents and:

1. **Extracts the file list** from the workspace layout section of the
   project detailed design. Every source file listed in the layout becomes
   a plan entry.

2. **Maps detailed design content to files.** For each file, the orchestrator
   identifies which sections of the detailed design describe its contents.
   A single file may draw from multiple topical sections (e.g.,
   `end_of_year_growth.rs` draws from Procedure Dispatch, Denomination
   Handling, CPI Cumulative Product, and Balance Bound).

3. **Extracts concrete specifications.** For each file, the orchestrator
   pulls: type definitions, function signatures, algorithm pseudocode,
   acceptance criteria, and test requirements from the relevant detailed
   design sections. These are included verbatim or lightly reformatted —
   the orchestrator does not paraphrase or interpret.

4. **Assigns milestones.** Files are grouped into milestones based on the
   build sequence in the project detailed design. Dependencies within a
   milestone are declared explicitly so the implementation orchestrator
   can determine serial vs. parallel dispatch.

5. **Specifies tests.** For each file, the orchestrator derives unit test
   specifications from the detailed design's acceptance criteria, numerical
   examples, and property-based test descriptions. Tests are written first
   (TDD) — the plan must specify tests before implementation for each file.

## Agent Construction

Spawn one agent per milestone (or per batch of files within a milestone if
the milestone is large). Each agent receives:

1. The complete detailed design documents (project-wide + component).
   No principles, no design docs, no requirements — the detailed design
   is self-contained.
2. The file list for its assigned milestone.
3. Instructions to produce the per-file plan entries in the format above.

Agents within the same milestone can run in parallel if they cover
non-overlapping file sets. Milestone ordering is sequential — the agent
producing M2's plan entries may need to reference M1's file list to declare
dependencies correctly.

## TDD Enforcement

Every file entry in the plan must include a **Tests (written first)** section.
The implementation orchestrator will instruct code-writing agents to write
tests before implementation. A plan entry without test specifications is
incomplete and must not be accepted.

Test specifications include:
- **Unit tests** for the inline `#[cfg(test)] mod tests` block
- **Golden-file test entries** (TOML fixture + expected results) for
  integration-level files
- **Property-based test cases** where the detailed design specifies them

## Validation

Before accepting the plan, the orchestrator verifies:

- Every source file in the workspace layout has a plan entry.
- Every plan entry has a `Tests (written first)` section.
- Every plan entry has an `Acceptance criteria` section.
- File dependencies within milestones form a DAG (no circular dependencies).
- Milestone ordering matches the build sequence in the project detailed design.

## Output

The plan is written to `implementation/<component>/plan.md`. It is a
disposable, per-cycle artifact — overwritten each release cycle. Git history
preserves prior versions.

The plan is immediately consumable by the
[implementation orchestrator](implementation-orchestrator.md) without
further transformation.

## Iteration

Plans are overwritten, not versioned. `plan.md` is always the current plan.

## Cleanup

The plan persists until the implementation cycle is complete. It may be
deleted after all code is written and tests pass — the detailed design
is the permanent record; the plan is the work order.

## Partitioning Strategy

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md).
Agents producing plan entries for independent milestones can run in parallel.
Within a milestone, batch file entries by directory where possible.

## Orchestration Notes

- Agents receive only the detailed design documents. No principles chain,
  no design documents, no requirements. The detailed design has already
  digested all of those into concrete decisions.
- Agents do not make design decisions. They reorganize existing content
  from topical arrangement to per-file arrangement.
- If a file's specification cannot be fully derived from the detailed design
  (a section is missing or ambiguous), the agent flags it as incomplete
  rather than inventing content.
- The plan must be producible by agents that have no domain knowledge beyond
  what the detailed design contains.
