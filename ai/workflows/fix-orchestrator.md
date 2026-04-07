# Fix Orchestrator

## Purpose

Orchestrates a targeted fix cycle. Consumes either a **review findings file**
or a **swarm fix file** and spawns one fix agent per affected file.

## Usage

Provide:
- **input file**: path to a review findings file (e.g.,
  `reviews/engine-engineer-design-1.md`) or a swarm fix file (e.g.,
  `reviews/fix-design-engine-streams.md`).
- **scope** (optional): limit fixes to a subset of files.

If missing, prompt: "Which findings or fix file should I use?"

## Input Formats

The orchestrator accepts two formats and detects which one it received:

### Review Findings

Produced by the [review orchestrator](review-orchestrator.md). Table format:

| ID | Location | Issue | Recommendation |
|---|---|---|---|

The orchestrator extracts Location, Issue (which quotes the artifact text),
and Recommendation (which describes the fix). **Skip QST findings** — these
require human decisions and cannot be applied mechanically. Collect skipped
QST findings into `reviews/questions-<slug>-<iteration>.md`.

Multi-location findings (continuation rows with blank ID) are grouped with
their parent finding.

### Swarm Fix Files

Produced by the [swarm orchestrator](swarm-orchestrator.md). Table format:

| Location | Quoted Text | Fix |
|---|---|---|

These are already in directly actionable form.

## Agent Construction

For each target file appearing in the input, spawn one agent with:

1. **Principles context:** assembled per
   [context-assembly.md](context-assembly.md), with inputs:
   - **personality** = none
   - **phase** = inferred from the input file's origin (e.g.,
     `engine-engineer-design-1` → `design`)
   - **subsystem** = inferred from the target file path when possible
2. **Your task:** Apply the fixes listed below to the file at `<path>`.
3. **Fixes:** *(all findings/fixes for this file path, verbatim from input)*
4. **Instructions:**
   - Read the file in full before making any changes.
   - For review findings: locate the quoted text from the Issue column and
     apply the Recommendation. If the quoted text cannot be found verbatim,
     use the Location to find the passage and apply the fix to the closest
     matching text.
   - For swarm fixes: replace the Quoted Text with the Fix exactly as
     specified.
   - Do not fix anything not listed. Do not refactor adjacent content.
   - Do not change document structure unless the fix explicitly requires it.
   - Write the updated file in place.
   - Produce a brief fix summary: one line per fix, stating what changed
     or why it was skipped.

## Partitioning Strategy

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md).
Small files with few fixes may be batched into a single agent.

## Output Format

Each fix agent writes to `reviews/fix-<file-slug>-<iteration>.md`:

```markdown
# Fix Summary: <file path>
**Input file:** <input file path>
**Date:** <YYYY-MM-DD>

## Applied Fixes

| Finding | Location | Change Made |
|---|---|---|
| <finding ID or —> | <location> | <one-line description> |

## Skipped Fixes

| Finding | Location | Reason Skipped |
|---|---|---|
| <finding ID or —> | <location> | <reason> |
```

After all agents complete, produce `reviews/fix-summary-<slug>-<iteration>.md`:
- Total files modified
- Total fixes actioned
- Total fixes skipped
- QST findings excluded (with IDs), if any

## Iteration

Scan `reviews/` for `fix-summary-<slug>-*.md`. Iteration = highest + 1, or 1.

## Orchestration Notes

- Spawn one agent per file in parallel.
- Fix agents do not re-review. They apply what the input says.
- After fixes, run the review orchestrator to verify — do not assume correctness.
- Fix agents never modify files outside their assigned path.
