# Fix Preparation

## Purpose

Converts review findings into actionable fix files that the
[fix orchestrator](fix-orchestrator.md) can consume. Review findings describe
problems; this workflow produces solutions.

## Input

A review findings file (e.g., `reviews/engine-engineer-design-1.md`) or a
review summary file (e.g., `reviews/summary-design-1.md`).

## Process

1. Read the findings file.
2. For each finding, determine the specific edit needed to resolve it.
3. Produce a fix file at `reviews/fixes-<slug>-<iteration>.md` in the
   fix orchestrator's required format:
   ```markdown
   ### <file path>

   | Location | Quoted Text | Fix |
   |---|---|---|
   | <section or line> | <exact text to replace> | <replacement text or instruction> |
   ```
4. QUESTION findings cannot be resolved without a human decision — collect
   them into `reviews/questions-<slug>-<iteration>.md` and exclude them
   from the fix file.
5. Present the fix file to the human for approval before launching the
   fix orchestrator.

## Output

- `reviews/fixes-<slug>-<iteration>.md` — actionable fix file
- `reviews/questions-<slug>-<iteration>.md` — findings requiring human decision (if any)
