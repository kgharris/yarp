# Review Orchestrator

> **STATUS: NEEDS REWORK.** This workflow is functional but has known issues
> with agent lane-drift and clunky iteration mechanics. See notes at bottom.

## Purpose

Orchestrates a multi-personality review cycle. Launches one agent per reviewer
personality, directs each to review their relevant artifacts, and collects
results as individual markdown files in `./reviews/`.

## Usage

Provide:
- **phase**: `requirements`, `design`, `implementation`, or `testing`
- **scope** (optional):
  - Omitted → derived from `git diff main`
  - `"full"` → all assigned artifacts in full

If phase is missing, prompt before proceeding.

**Slug** = phase name, lowercased.
**Iteration** = highest N in `reviews/summary-<slug>-*.md` + 1, or 1.

## Scope Generation

For git-diff scope: generate a file list from diffs, write to
`reviews/file-list-<slug>.md`. All agents use this to filter their artifacts.

## Reviewer Assignments

Launch one agent per personality:

- `ai/personalities/product-owner.md`
- `ai/personalities/principal-engineer.md`
- `ai/personalities/ux-engineer.md`
- `ai/personalities/engine-engineer.md`
- `ai/personalities/ux-qa-engineer.md`
- `ai/personalities/engine-qa-engineer.md`
- `ai/personalities/financial-domain-reviewer.md`
- `ai/personalities/db-engineer.md`

Each agent receives context assembled per
[context-assembly.md](context-assembly.md), with inputs:
- **personality** = the reviewer's personality file
- **phase** = the review phase
- **subsystem** = determined by the personality's domain (none for Forest agents)

The assembled context is delivered inline in the agent's prompt — agents
do not fetch principles themselves. This is the primary control for
preventing lane drift.

## Output Format

Each reviewer writes: `reviews/<personality-file-stem>-<slug>-<iteration>.md`

```markdown
# Review: <Personality Name>
**Phase:** <phase> | **Iteration:** <iteration> | **Date:** <YYYY-MM-DD>

## Summary
2-4 sentence assessment.

## Findings

| ID | Severity | Location | Issue | Recommendation |
|---|---|---|---|---|
| <ABBREV>-<SLUG>-<N> | CRITICAL/MAJOR/MINOR/QUESTION | <file:section> | What is wrong | What should change |

## Passed Checks
Brief list of areas reviewed and found correct.

## Out of Scope
Anything intentionally not covered.
```

Severity levels:
- **CRITICAL** — Wrong, inconsistent, or missing; will cause incorrect behavior or block implementation.
- **MAJOR** — Significant gap or ambiguity; will cause implementation divergence.
- **MINOR** — Small gap or wording issue; won't block implementation.
- **QUESTION** — Requires a decision before it can be assessed.

## Summary File

After all reviewers complete, produce `reviews/summary-<slug>-<iteration>.md`:

- Finding-count table by severity and reviewer.
- **Consolidated Findings** clustering related findings from different reviewers.
  Each gets a meta-tag `META-<SLUG>-<N>` with constituent finding IDs.
  Severity inherits the most severe constituent rating.
- Priority recommendation for what to address first.

## Iteration 2+

Re-reviews are driven by the orchestrator diffing prior findings against
current artifacts — not by asking agents to read their own prior output.

## Partitioning Strategy

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md).

## Orchestration Notes

- Launch all reviewers in parallel.
- Reviewers do not read each other's output.
- Each reviewer's perspective is independent.

---

## Known Issues (Rework Needed)

- **Lane drift**: Agents produce findings outside their personality's domain
  despite explicit constraints. The structured table format above (replacing
  prose findings) is intended to help but hasn't been validated yet.
- **Iteration mechanics**: The old model asked agents to re-review their own
  prior output, which compounded drift. The new model (orchestrator-driven
  diffing) is specified above but not yet implemented.
- **Scope from git diff**: Underspecified — needs clearer rules about what
  agents read in full vs. skim.
