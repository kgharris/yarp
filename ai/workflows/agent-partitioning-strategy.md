# Agent Partitioning Strategy

This document defines how orchestrators partition work across agents to
manage token usage and maximize parallelism. All orchestrators must apply
this strategy before spawning agents.

Partitioning decisions depend on agent altitude (see
[principles.md](principles.md)). An agent that must see the whole system
cannot be split the same way as one that operates on a single file.

---

## Altitude Constraints

Altitude overrides size-based rules. Determine the agent's altitude
first, then apply the appropriate partitioning:

**Forest** agents need cross-system visibility. Do not partition their
input — give them the full file list. Forest agents skim rather than
deep-read, so breadth matters more than per-file depth. If the total
content exceeds token limits, reduce per-file depth (summaries, headers
only) rather than splitting into multiple forest agents. Splitting a
forest agent defeats its purpose: it can only find cross-cutting issues
if it sees the whole system.

**Branch** agents span one domain across multiple subsystems. Partition
by domain boundary, not by file size. A branch agent must see all files
in its domain together. Only split a branch agent if the domain itself
has natural sub-domains that can be reviewed independently.

**Leaf** agents operate within one subsystem at full depth. Size-based
partitioning applies here — leaf agents are the primary target of the
file-batching rules below.

---

## File Discovery

Before spawning agents, the orchestrator:
1. Resolves the full file list from the scope (glob, directory list,
   plan entries, or personality artifact lists).
2. Reads file sizes from the filesystem in the same pass (e.g.,
   `find <scope> -name "*.md" -printf "%s %p\n"` or equivalent).

---

## Size-Based Partitioning (Leaf Agents)

Partition files into agent assignments based on actual size. These rules
apply to leaf-altitude agents and to orchestrators (swarm, fix,
implementation) where each agent operates on individual files:

- **Small files** (< 10 KB): batch up to 10 files per agent, grouped by
  directory where possible to minimize context switching.
- **Large files** (>= 10 KB): one agent per file.
- **Mixed directories**: large files get their own agent; remaining small
  files in the same directory are batched together.

---

## Threshold Overrides

The size thresholds above are defaults. An orchestrator's input document
(swarm definition, implementation plan, etc.) may override them:

```markdown
## Partitioning
small_file_threshold_kb: 20
max_batch_size: 5
```

If not specified, defaults apply.

---

## Dependency-Ordered Partitioning

When work items have dependencies (e.g., cross-subsystem implementation),
partitioning must respect dependency order. Agents within the same
dependency tier can run in parallel; downstream tiers wait until upstream
agents complete or stubs are in place.

Size-based batching still applies within each tier.

---

## Prompt Size

Keep each agent's prompt as small as possible. Include only:
- The file path(s) assigned to this agent
- The criteria, specification, or instructions relevant to those files
- Shared context that the agent needs to evaluate its assignment

Do not include the full orchestrator definition, full input document, or
material for files assigned to other agents.
