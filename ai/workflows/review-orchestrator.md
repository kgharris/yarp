# Review Orchestrator

## Purpose

Orchestrates a multi-personality review cycle. Launches one agent per reviewer
personality, directs each to review their relevant artifacts, and collects
results as individual markdown files in `./reviews/`.

## Usage

Provide:
- **phase**: `requirements`, `design`, `implementation`, or `testing`
- **sub-phase** (implementation only): `detailed-design`, `plan`, or `code`
- **scope** (optional):
  - Omitted → derived from `git diff main`
  - `"full"` → all assigned artifacts in full

If phase is missing, prompt before proceeding. If phase is `implementation`
and sub-phase is missing, prompt before proceeding.

**Slug** = `<phase>` for non-implementation phases, or
`<phase>-<sub-phase>` for implementation (e.g., `implementation-detailed-design`).
**Iteration** = highest N in `reviews/summary-<slug>-*.md` + 1, or 1.

### Implementation Sub-Phases

The implementation phase has three distinct sub-phases, each reviewing
different artifacts against a different baseline:

| Sub-phase | Artifacts under review | Compared against | Key question |
|-----------|----------------------|------------------|--------------|
| `detailed-design` | `implementation/<component>/detailed-design.md` | `design/` docs | Does the detailed design correctly realize the design? |
| `plan` | `implementation/<component>/plan.md` | `implementation/<component>/detailed-design.md` | Are file specs complete, tests specified, dependencies correct? |
| `code` | Source files in `src/` | `implementation/<component>/plan.md` | Does the code satisfy the plan's specifications and acceptance criteria? |

The sub-phase determines what agents are told to review and what counts as a
finding. Context assembly uses `phase = implementation` for all three —
the same principles and personality focus apply. The sub-phase is
additional routing provided by the orchestrator in the agent prompt.

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

## Finding Quality Bar

A finding must clear this bar or it should not be raised. Reviewers are not
evaluated on finding count — a review with zero findings and a thorough Passed
Checks section is a valid and valuable output.

**The test:** if this finding is ignored, will an implementer produce incorrect
output, violate an architectural boundary, or be unable to proceed? If no, do
not raise it.

**Do not raise findings about:**

- Divergence between a principles file and a spec — principles are orientation,
  not contracts.
- Missing error messages or error type variants — error-handling.md declares a
  format; specific messages are implementation concerns.
- Implementation details — JSON serialization shape, UUID strategy, serde
  attributes, Rust type signatures, specific library choices.
- Forward-looking content in orientation documents that is not part of MVP.
- Behavior obvious from the data structure — tree traversal order, derivation
  of values that are available in the data model.
- Missing clarifying notes when the meaning is clear from context.
- Unreachable code paths that exist as defensive design.
- The absence of new types, enums, or variants when string labels or existing
  composition mechanisms suffice.

**Severity must be proportionate.** CRT means wrong, inconsistent, or missing
in a way that will cause incorrect behavior or block implementation. A
formatting preference or a suggestion for optional documentation is MIN at
most. Inflated severity wastes triage time.

## Output Format

Each reviewer writes: `reviews/<personality-file-stem>-<slug>-<iteration>.md`

```markdown
# Review: <Personality Name>
**Phase:** <phase> | **Iteration:** <iteration> | **Date:** <YYYY-MM-DD>

## Summary
2-4 sentence assessment.

## Findings

| ID | Location | Issue | Recommendation |
|---|---|---|---|
| <a id="<ID>"></a><ID> | <file>:<heading path or line> | Problem description quoting artifact text | What should change |

## Passed Checks
Brief list of areas reviewed and found correct.

## Out of Scope
Anything intentionally not covered.
```

Sort the table by severity: CRT first, then MAJ, MIN, QST.

Column definitions:

- **ID** — `<ABBREV>-<SLUG>-<SEV>-<NN>`. Each ID cell includes an HTML
  anchor (`<a id="..."></a>`) so the summary file can deep-link to it.
  - ABBREV = personality's initials (PO, PE, UXE, EE, UXQA, EQA, FDR, DBE).
  - SLUG = short kebab-case topic.
  - SEV = severity code:
    - CRT — Wrong, inconsistent, or missing; will cause incorrect behavior or block implementation.
    - MAJ — Significant gap or ambiguity; will cause implementation divergence.
    - MIN — Small gap or wording issue; won't block implementation.
    - QST — Requires a decision before it can be assessed.
  - NN = two-digit sequence number (01–99), scoped per severity level
    (CRT-01, CRT-02, …; MAJ-01, MAJ-02, … each start at 01).
- **Location** — File path and heading path or line number. Use the deepest
  relevant heading to pinpoint the passage. Examples:
  `requirements/engine/timeline.md: Forward Sweep > Boundary Conditions`,
  `design/engine/projection.md:L42`.
  When a finding spans multiple locations, use additional rows with ID blank:

  | ID | Location | Issue | Recommendation |
  |---|---|---|---|
  | <a id="EE-example-CRT-01"></a>EE-example-CRT-01 | design/engine/projection.md:L42 | description quoting first location | specific edit for this location |
  | | design/engine/streams.md: Stream Construction | quote from second location | specific edit for this location (or blank if identical to above) |
- **Issue** — Describe the problem and quote the specific artifact text that
  is wrong or missing. The quote grounds the finding to a concrete passage
  and enables the [fix orchestrator](fix-orchestrator.md) to apply fixes
  without re-analysis.
- **Recommendation** — What should change. Be specific enough that a fix
  can be drafted without re-reading the surrounding context.

## Orchestrator Triage

After all reviewers complete and before producing the summary, the orchestrator
triages every finding. The orchestrator has access to the project's feedback
memory and prior review history. Its job is to filter noise so the user reviews
only findings that would cause a wrong implementation or block progress.

**Reject a finding if it matches any of these patterns:**

- Flags divergence between a principles file and a spec — principles are
  aspirational orientation, not contractual. Divergence is never a finding.
- Asks for exhaustive error message catalogues — error-handling.md declares a
  format. Specific messages are implementation concerns.
- Flags implementation details as design gaps — JSON serialization shape, UUID
  generation strategy, serde attribute choices, specific Rust type signatures.
- Flags forward-looking content in orientation documents (architecture.md,
  principles files) as if it should be MVP — it is intentionally forward-looking.
- Flags behavior that is obvious from the data structure — tree traversal order,
  year list derivation from stream points, implicit uniqueness constraints.
- Asks for clarifying notes on things already clear from context.
- Flags unreachable code paths that exist as defensive design.
- Flags the absence of design coverage for FUT-tagged requirements — the scope
  gate should have caught this, but if it leaked through, reject it here.
- Describes a data-driven architecture concern that invents new types where
  string labels or existing composition would suffice.
- Inflates severity — a finding that is at most MIN should not be rated CRT or
  MAJ. The orchestrator may downgrade severity when the agent's rating is
  disproportionate to the actual impact.

**The triage bar:** would this finding, if ignored, cause an implementer to
produce incorrect output, violate an architectural boundary, or be unable to
proceed? If no, reject it.

Rejected findings are listed in a **Rejected by Orchestrator** appendix in the
summary file with a one-line reason each. The user can review them if desired
but is not expected to.

## Summary File

After triage, produce `reviews/summary-<slug>-<iteration>.md`:

- Finding-count table by severity and reviewer (active findings only).
- **Consolidated Findings** table clustering related active findings from
  different reviewers. Sort by severity (CRT first, then MAJ, MIN, QST):

  | ID | Citation | Issue | Recommendation |
  |---|---|---|---|
  | META-<SLUG>-<SEV>-<NN> | <finding-ID> | Problem description | What should change |
  | | <finding-ID> | … or blank if agrees with above | … or blank |

  - **ID** — `META-<SLUG>-<SEV>-<NN>`. Same SEV/NN rules as agent findings.
    Severity inherits the most severe constituent rating. Appears only on
    the first row of each group.
  - **Citation** — One finding ID per row, linked to the agent's review file
    with a markdown anchor: `[EE-boundary-CRT-01](engine-engineer-design-1.md#EE-boundary-CRT-01)`.
  - **Issue** and **Recommendation** — same rules as agent findings.

  Each cited finding gets its own row. The first row carries the META ID;
  subsequent rows leave ID blank and carry only their citation, issue, and
  recommendation. When a cited agent agrees entirely with the row above it,
  leave Issue and Recommendation blank — the citation alone signals agreement.
  When agents agree on the issue but assigned different severities, attribute
  the Issue and Recommendation to the agent with the highest severity and
  leave the others blank.

  Example:

  | ID | Citation | Issue | Recommendation |
  |---|---|---|---|
  | META-boundary-CRT-01 | [EE-boundary-CRT-01](engine-engineer-design-1.md#EE-boundary-CRT-01) | stream lacks null check for year 0 | add guard clause at stream head |
  | | [PE-boundary-CRT-02](principal-engineer-design-1.md#PE-boundary-CRT-02) | stream produces wrong value at boundary | redesign stream to internalize boundary |
  | | [EE-boundary-MAJ-04](engine-engineer-design-1.md#EE-boundary-MAJ-04) | | |

- **Rejected by Orchestrator** appendix — a flat table of findings rejected
  during triage:

  | Agent Finding | Reason |
  |---|---|
  | EE-xxx-MIN-01 | Implementation detail (serde convention) |
  | PE-xxx-CRT-02 | Principles divergence, not a spec issue |

- Priority recommendation for what to address first.

## Partitioning Strategy

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md).

## Orchestration Notes

- Launch all reviewers in parallel.
- Reviewers do not read each other's output.
- Each reviewer's perspective is independent.

