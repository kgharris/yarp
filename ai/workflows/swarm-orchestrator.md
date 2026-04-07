# Swarm Search Orchestrator

## Purpose

Orchestrates a targeted search across the codebase. Spawns one agent per file,
each checking a single document against criteria defined in a **swarm definition
file**, and collects results into a single report.

The orchestrator manages agents. It does not know what to search for — that
comes from the swarm definition.

## Usage

Provide:
- **swarm definition**: path to a swarm definition file.

If missing, prompt: "Which swarm definition should I use?"

A **scope override** may replace the definition's scope for one run.

## Swarm Definition Contract

The swarm definition must contain:

- **Objective** — what this sweep is looking for.
- **Scope** — which files to search (globs, directory list, or "all docs"). May include exclusions.
- **Search Criteria** — the categories of findings agents look for.
- **Context for Agents** — concise reference material agents need. Must be short enough to include verbatim in each agent prompt.
- **Output Filename** — path under `./reviews/` for the merged report.

## Agent Construction

For each file in scope, spawn one agent with:

1. **Principles context:** assembled per
   [context-assembly.md](context-assembly.md), with inputs:
   - **personality** = none (swarm agents are criteria-driven)
   - **phase** = determined from the swarm definition's scope (if applicable)
   - **subsystem** = none
2. **Your task:** Review the file at `<path>` for issues described below.
3. **Search criteria:** *(verbatim from definition)*
4. **Context:** *(verbatim from definition)*
5. **Instructions:**
   - Read the assigned file in full.
   - For each finding, record:
     - **Location** (line number or section name)
     - **Quoted text** — exact text to be replaced
     - **Fix** — replacement text or instruction
   - Format findings as a table:
     ```markdown
     | Location | Quoted Text | Fix |
     |---|---|---|
     | Line 42 | old text | new text |
     ```
   - If the file has no findings, report it as clean.
   - Do not fix files. Report only.

## Partitioning Strategy

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md) to the
resolved file list before spawning agents. The swarm definition may override
default thresholds.

## Output Format

### Per-Agent Fix Files

Each agent with findings produces a fix file at `reviews/fix-<file-slug>.md`:

```markdown
### <file path>

| Location | Quoted Text | Fix |
|---|---|---|
| Line 42 | exact text to replace | replacement text |
```

`<file-slug>` = source path with slashes replaced by dashes, `.md` stripped.

### Merged Summary

All findings collected into the file specified by the definition's Output Filename:

```markdown
# <Objective — short title>
**Date:** <YYYY-MM-DD>
**Scope:** <files searched>
**Definition:** <swarm definition file path>

## Summary
<N> files searched, <M> files with findings, <K> total findings.
<M> fix files produced under reviews/.

## Findings by File

### <file path>
**Fix file:** `reviews/fix-<file-slug>.md`

| Location | Quoted Text | Fix |
|---|---|---|
| Line 42 | exact old text | replacement text |

### <file path>
...

## Clean Files
- <file path>
- ...
```

## Orchestration Notes

- Spawn agents in parallel — they are independent.
- Each agent reads exactly one file. No cross-referencing.
- Keep agent prompts small: search criteria, context, and file path only.
- After all agents return, merge results sorted by file path.
- If no findings exist, write the output file with an empty findings section.
