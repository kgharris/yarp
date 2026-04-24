# Implementation Orchestrator

## Purpose

Orchestrates the code-writing phase. Consumes an **implementation plan** and
spawns one agent per file or logical unit to write the actual code.

The orchestrator does not produce implementation plans. Plans are the output of
the [implementation plan orchestrator](implementation-plan-orchestrator.md) and
exist before this orchestrator is invoked.

## Usage

Provide:
- **implementation plan**: path to a plan file.
- **scope** (optional): limit to a subset of files or components.

If missing, prompt: "Which implementation plan should I use?"

## Implementation Plan Format

The plan is organized by milestone, with one section per file. Each file entry
contains enough detail to write code without referring back to design documents.
The format is defined by the
[implementation plan orchestrator](implementation-plan-orchestrator.md#plan-format).

```markdown
### `<file path>`

**Realizes:** <detailed design section(s) this file implements>
**Purpose:** <what this file does>
**Depends on:** <other file paths in this milestone, or "none">

**Specification:**
- Types, traits, functions, or components to define
- Behavioral requirements
- Constraints

**Tests (written first):**
- Unit tests for the inline test module
- Each test: name, inputs, expected output

**Acceptance criteria:**
- <observable conditions for this file being done>
```

## Agent Construction

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md) to
batch file entries into agent assignments. Small related files (e.g., entity
type modules in the same directory) are batched to a single agent. Large or
complex files get their own agent. Batching respects declared dependencies —
files in the same batch must not depend on each other unless the batch
preserves their ordering.

Each agent receives:

1. **No principles context.** Implementation agents are plan-driven. The plan
   contains all specifications, type signatures, and constraints needed to
   write the code. Agents do not receive the principles chain, design
   documents, or requirements — the detailed design has already digested
   those into the plan's concrete specifications.
2. **Your task:** Write the following file(s) as specified below.
3. **Specification:** *(verbatim plan entries for all files in this batch)*
4. **Tests:** *(verbatim "Tests (written first)" sections for all files)*
5. **Instructions:**
   - **Write tests first.** For each file, implement the unit tests specified
     in the plan's test section before writing the implementation code. Tests
     go in the inline `#[cfg(test)] mod tests` block at the bottom of each
     file.
   - Read any existing file at a target path before writing.
   - Write code that satisfies the specification completely.
   - Do not write code for files outside your assignment.
   - Include inline comments only where logic is not self-evident.
   - Produce a brief implementation summary per file: what was created, any
     specification items that could not be satisfied, and why.

## Dependency Order

When a plan spans multiple subsystems, launch in dependency order:

1. **Engine** — pure computation, no dependencies
2. **DB** — persistence, depends on engine data structures
3. **Controller** — mediates engine and UX, depends on both
4. **UX** — presentation, depends on controller interface

Do not launch downstream agents until upstream files exist or the plan
provides stubs.

## Output Format

Each agent writes to `reviews/impl-<file-slug>-<iteration>.md`:

```markdown
# Implementation Summary: <file path>
**Plan:** <plan path>
**Date:** <YYYY-MM-DD>

## Created

| Item | Description |
|---|---|
| <type/function/component> | <one-line description> |

## Deferred

| Specification Item | Reason |
|---|---|
| <item> | <reason> |
```

After all agents complete, produce `reviews/impl-summary-<plan-slug>-<iteration>.md`:
- Total files created or modified
- Total specification items satisfied
- Total items deferred
- Dependency order violations (if any)
- Suggested next steps

## Iteration

Scan `reviews/` for `impl-summary-<plan-slug>-*.md`. Iteration = highest + 1, or 1.

## Partitioning Strategy

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md).
Use dependency-ordered partitioning when spanning subsystems.

## Orchestration Notes

- Agents within the same milestone run in parallel (respecting declared
  dependencies). Cross-milestone agents run sequentially.
- Cross-subsystem agents respect dependency order.
- Agents do not re-design. They write what the plan says.
- Agents write tests first, then implementation. Both are in the same file
  (inline `#[cfg(test)] mod tests`).
- After implementation, run the review orchestrator with `phase=implementation`
  to verify against design.
- Agents never modify files outside their assigned path.
