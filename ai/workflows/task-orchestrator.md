# Task Orchestrator

## Purpose

Launches a single personality agent to perform a specific, focused task. Use this
when the job does not require a full review swarm — e.g., drafting a section,
answering a question, analyzing one artifact, or investigating a specific concern.

## Usage

Provide:
- **personality** — which reviewer to use (e.g., `engine-engineer`, `product-owner`)
- **phase** — `requirements`, `design`, `implementation`, or `testing`
- **task** — a clear description of what the agent should do

If any of these are missing, prompt before proceeding.

**Output slug** = `<personality-file-stem>-task-<YYYY-MM-DD>`

## Context Assembly

Assemble the agent's context per [context-assembly.md](context-assembly.md), with:
- **personality** = the named personality file
- **phase** = the named phase
- **subsystem** = determined by the personality's domain (none for Forest agents)

Deliver all context inline in the agent's prompt.

## Agent Prompt Structure

```
<assembled context — inline, per context-assembly.md>

---

## Your Task

<task description, verbatim from the user>

## Output

Write your output to: reviews/<output-slug>.md

Use the following format:

# Task Output: <Personality Name>
**Phase:** <phase> | **Task:** <one-line summary> | **Date:** <YYYY-MM-DD>

## Summary
2–4 sentence overview of what you did and what you found.

## Output
<task-specific content — prose, table, draft, analysis, or findings as appropriate>

## Notes
Anything that fell outside the task scope but warrants attention.
```

## Orchestration Notes

- Launch one agent only.
- No summary file — the single output file is the result.
- If the task produces findings that warrant follow-up, the user may promote
  them into a full review or fix cycle.
