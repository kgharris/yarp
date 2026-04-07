# Implementation Orchestrator

## Purpose

Orchestrates the code-writing phase. Consumes an **implementation plan** and
spawns one agent per file or logical unit to write the actual code.

The orchestrator does not produce implementation plans. Plans are the output of
the requirements -> design -> implementation planning workflow and exist before
this orchestrator is invoked.

## Usage

Provide:
- **implementation plan**: path to a plan file.
- **scope** (optional): limit to a subset of files or components.

If missing, prompt: "Which implementation plan should I use?"

## Implementation Plan Format

One section per file with enough detail to write code without referring back to
design documents:

```markdown
### <file path>

**Realizes:** <design document section(s) this file implements>
**Purpose:** <what this file does>

**Specification:**
- Types, traits, functions, or components to define
- Behavioral requirements from the design
- Dependencies on other files in this plan
- Constraints from implementation principles

**Acceptance criteria:**
- <criteria derived from requirements>
```

## Agent Construction

For each file entry, spawn one agent with:

1. **Principles context:** assembled per
   [context-assembly.md](context-assembly.md), with inputs:
   - **personality** = none (implementation agents are plan-driven)
   - **phase** = `implementation`
   - **subsystem** = determined from the target file path (engine, db,
     controller, ux)
2. **Your task:** Write the file at `<path>` as specified below.
3. **Specification:** *(verbatim from plan for this file)*
4. **Instructions:**
   - Read any existing file at the target path before writing.
   - Write code that satisfies the specification completely.
   - Follow all coding standards from the principles context.
   - Do not write code for files outside your assignment.
   - Do not write tests — that is a separate concern.
   - Include inline comments only where logic is not self-evident.
   - Produce a brief implementation summary: what was created, any
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

- Agents within the same subsystem run in parallel.
- Cross-subsystem agents respect dependency order.
- Agents do not re-design. They write what the plan says.
- After implementation, run the review orchestrator with `phase=implementation`
  to verify against design.
- Agents never modify files outside their assigned path.
