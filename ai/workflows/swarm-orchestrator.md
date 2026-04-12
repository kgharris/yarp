# Swarm Orchestrator

## Purpose

Orchestrates parallel work across a set of files or subdomains. Spawns multiple
agents — each handling an independent unit of work — and collects results into a
summary report.

The orchestrator manages agents. What they do and how they partition work is
determined by the **swarm definition file**.

## Usage

Provide:
- **swarm definition**: path to a swarm definition file.

If missing, prompt: "Which swarm definition should I use?"

A **scope override** may replace the definition's scope for one run.

---

## Swarm Modes

Every swarm definition declares a **mode** that determines agent type, context
assembly, and output format. Two modes are supported:

### `search` mode (default)

Criteria-driven audit. One agent per file. Each agent checks a single document
against fixed search criteria and reports findings. No personality context — agents
are criteria-driven, not role-driven.

Use when: looking for violations, inconsistencies, or antipatterns across many files.

### `personality` mode

Personality-driven work. One agent per partition (subdomain). Each agent receives
the full personality context assembled per
[context-assembly.md](context-assembly.md) and performs a specified task on its
assigned subdomain — drafting a doc, analyzing a domain, producing a deliverable.

Use when: each subdomain needs deep, expert work: writing design docs, drafting
requirements, producing structured analysis.

---

## Swarm Definition Contract

All swarm definitions must contain:

- **Mode** — `search` or `personality`. Defaults to `search` if omitted.
- **Objective** — what this swarm accomplishes.
- **Output Filename** — path under `./reviews/` for the merged summary report.

### Additional fields for `search` mode

- **Scope** — which files to search (globs, directory list, or "all docs"). May
  include exclusions.
- **Search Criteria** — the categories of findings agents look for.
- **Context for Agents** — concise reference material agents need. Must be short
  enough to include verbatim in each agent prompt.

### Additional fields for `personality` mode

- **Personality** — which personality to use (e.g., `engine-engineer`). Must
  match a file in `ai/personalities/`.
- **Phase** — `requirements`, `design`, `implementation`, or `testing`.
- **Partitions** — list of named subdomains. Each partition has:
  - `name` — short identifier for this partition (used in filenames and headings).
  - `task` — what the agent must do. Written as a directive ("Draft the design
    doc for the timeline module…").
  - `output` — path where the agent writes its artifact.
  - `scope` (optional) — files or globs the agent should read. If omitted, the
    agent uses its personality's standard artifact list for the phase.
- **Context for Agents** (optional) — shared background all agents in this swarm
  need beyond their assembled personality context.

---

## Agent Construction

### `search` mode agents

For each file in scope, spawn one agent with:

1. **Principles context:** assembled per [context-assembly.md](context-assembly.md),
   with inputs:
   - **personality** = none (agents are criteria-driven)
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

### `personality` mode agents

For each partition in the definition, spawn one agent with:

1. **Principles context:** assembled per [context-assembly.md](context-assembly.md),
   with inputs:
   - **personality** = the definition's Personality value
   - **phase** = the definition's Phase value
   - **subsystem** = determined by the personality's domain
2. **Scope:** if the partition specifies a scope, list those files explicitly.
   If not, instruct the agent to use its standard artifact list for the phase.
3. **Shared context:** *(verbatim from definition's Context for Agents, if present)*
4. **Your task:** *(verbatim from the partition's task field)*
5. **Output:** Write your artifact to `<partition output path>`.
6. **Instructions:**
   - Read all files in your scope before writing.
   - Produce the artifact described in the task.
   - Write the artifact to the specified output path.
   - Do not produce a findings table — produce the requested deliverable.

---

## Partitioning Strategy

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md) before
spawning agents. The swarm definition may override default thresholds.

For `search` mode: partition by file, following the file-batching rules.

For `personality` mode: each partition in the definition is one agent. Do not
split or merge partitions — the partition structure is authoritative. Within a
partition, the agent reads its scope files independently; no size-based batching
applies to the agent's reading.

---

## Output Format

### `search` mode

#### Per-Agent Fix Files

Each agent with findings produces a fix file at `reviews/fix-<file-slug>.md`:

```markdown
### <file path>

| Location | Quoted Text | Fix |
|---|---|---|
| Line 42 | exact text to replace | replacement text |
```

`<file-slug>` = source path with slashes replaced by dashes, `.md` stripped.

#### Merged Summary

All findings collected into the file specified by Output Filename:

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

### `personality` mode

#### Per-Agent Artifacts

Each agent writes its artifact directly to the path specified in its partition's
`output` field. These are first-class project files (design docs, requirements
sections, analysis reports), not reviews output.

#### Merged Summary

A summary report is written to the file specified by Output Filename:

```markdown
# <Objective — short title>
**Date:** <YYYY-MM-DD>
**Personality:** <personality> | **Phase:** <phase>
**Definition:** <swarm definition file path>

## Summary
<N> partitions executed, <M> artifacts produced.

## Artifacts Produced

| Partition | Output Path | Status |
|---|---|---|
| <name> | <output path> | written |
| <name> | <output path> | written |

## Notes
<Any cross-partition observations the orchestrator collected. Omit if none.>
```

---

## Orchestration Notes

- Spawn agents in parallel — they are independent within a swarm.
- **`search` mode:** each agent reads exactly one file. No cross-referencing.
  Keep agent prompts small: search criteria, context, and file path only.
- **`personality` mode:** each agent works its full partition. Agents should not
  cross partition boundaries — if a partition's task requires content from another
  partition, note the dependency in the swarm definition and sequence those
  partitions explicitly.
- After all agents return, merge results (findings for search; artifact list for
  personality) sorted by file path or partition name.
- If no findings exist (search mode), write the output file with an empty findings
  section.
- If an artifact was not produced (personality mode), record the partition as
  `skipped` in the summary and note the reason.
