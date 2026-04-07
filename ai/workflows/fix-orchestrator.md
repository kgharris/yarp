# Fix Orchestrator

## Purpose

Orchestrates a targeted fix cycle. Consumes a **fix file** — specific,
actionable edits per document — and spawns one fix agent per affected file.

This orchestrator does not consume review findings directly. Review findings
describe problems; fix files describe solutions. To convert review findings
into fix files, see [fix-preparation.md](fix-preparation.md).

## Usage

Provide:
- **fix file**: path to a fix file (e.g., `reviews/fixes-design-1.md`).
- **scope** (optional): limit fixes to a subset of files.

If missing, prompt: "Which fix file should I use?"

If no file is specified, scan `reviews/` for files matching `fixes-*-*.md`,
pick the highest iteration number, and use that.

## Fix File Format

One section per affected file with actionable edits:

```markdown
### <file path>

| Location | Quoted Text | Fix |
|---|---|---|
| <section or line> | <exact text to replace> | <replacement text or instruction> |
```

Fix files are produced by:
- **Swarm agents** directly (as part of swarm output)
- **Fix preparation workflow** (converting review findings into fix files)

## Agent Construction

For each file in the fix file, spawn one agent with:

1. **Principles context:** assembled per
   [context-assembly.md](context-assembly.md), with inputs:
   - **personality** = none
   - **phase** = inferred from the fix file's origin (e.g., `fixes-design-1`
     → `design`)
   - **subsystem** = inferred from the target file path when possible
2. **Your task:** Apply the fixes listed below to the file at `<path>`.
3. **Fixes:** *(verbatim from fix file for this path)*
4. **Instructions:**
   - Read the file in full before making any changes.
   - Apply each fix exactly as specified.
   - Do not fix anything not listed. Do not refactor adjacent content.
   - Do not change document structure unless the fix explicitly requires it.
   - Write the updated file in place.
   - Produce a brief fix summary: one line per fix, stating what changed or why it was skipped.

## Partitioning Strategy

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md).
Small files with few fixes may be batched into a single agent.

## Output Format

Each fix agent writes to `reviews/fix-<file-slug>-<iteration>.md`:

```markdown
# Fix Summary: <file path>
**Fix file:** <fix file path>
**Date:** <YYYY-MM-DD>

## Applied Fixes

| Location | Change Made |
|---|---|
| <location> | <one-line description> |

## Skipped Fixes

| Location | Reason Skipped |
|---|---|
| <location> | <reason> |
```

After all agents complete, produce `reviews/fix-summary-<slug>-<iteration>.md`:
- Total files modified
- Total fixes actioned
- Total fixes skipped

## Iteration

Scan `reviews/` for `fix-summary-<slug>-*.md`. Iteration = highest + 1, or 1.

## Orchestration Notes

- Spawn one agent per file in parallel.
- Fix agents do not re-review. They apply what the fix file says.
- After fixes, run the review orchestrator to verify — do not assume correctness.
- Fix agents never modify files outside their assigned path.
