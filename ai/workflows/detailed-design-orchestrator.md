# Detailed Design Orchestrator

## Purpose

Orchestrates the creation of a detailed design spec for a component. Runs three
phases: draft generation (multiple parallel agents produce independent specs),
evaluation (multiple principal engineers compare drafts against design docs and
rank them), and convergence (one principal engineer distills evaluation results
into a single set of instructions).

The output is a convergence doc that the user refines collaboratively into the
final `detailed-design.md`. The spec captures implementation-level decisions — data
structures, algorithms, type signatures, and their relationships — at a level
below `design/` but above individual source files. Per-file implementation
plans are derived from the spec separately.

## Usage

Provide:
- **component**: which subsystem to spec (`engine`, `db`, `controller`, `ux`)
- **scope** (optional): limit to a subset of the component

If component is missing, prompt before proceeding.

**Paths:**
- Draft specs: `implementation/<component>/draft-<N>.md`
- Evaluation reports: `reviews/pe-<component>-specs-<N>.md`
- Convergence instructions: `reviews/converge-<component>-specs.md`
- Final spec (produced collaboratively, not by this orchestrator):
  `implementation/<component>/detailed-design.md`

## Phase 1 --- Draft Generation

Spawn N agents (default 5) of the component's engineer personality in parallel.
Each drafts an independent detailed design spec for the component.

Each agent receives:
1. Context assembled per [context-assembly.md](context-assembly.md), with:
   - **personality** = the component's engineer personality
   - **phase** = `implementation`
   - **subsystem** = the component
2. A distinct focus angle. Assign each agent a different emphasis so drafts
   explore the design space rather than converging prematurely:
   - Agent 1: bottom-up, type-first (crate structure, core types, build upward)
   - Agent 2: trait-driven, testability-first (contracts, validation, test strategy)
   - Agent 3: algorithm core outward (central computation loop, then supporting structures)
   - Agent 4: data flow and persistence boundary (in-memory graph, generate, CLI)
   - Agent 5: incremental delivery milestones (thinnest vertical slice, layered growth)
3. The full set of design documents relevant to the component (delivered inline,
   not by reference).
4. Instructions to write the draft to `implementation/<component>/draft-<N>.md`.

All agents run in parallel. They do not see each other's output.

## Phase 2 --- Evaluation

After all drafts complete, spawn M principal engineers (default 3) in parallel.
Each reads all N drafts and all relevant design documents, then produces an
independent evaluation report.

Each evaluator receives:
1. Context assembled per [context-assembly.md](context-assembly.md), with:
   - **personality** = `principal-engineer`
   - **phase** = `implementation`
   - **subsystem** = none (Forest altitude)
2. All N draft specs (by path).
3. All design documents relevant to the component (by path).
4. Instructions to:
   - Define objective evaluation criteria
   - Assess each draft against every criterion
   - Rank all drafts with justification
   - Identify which drafts' strengths are complementary vs. incompatible
   - Recommend how to derive one spec from the N alternatives
   - Write the report to `reviews/pe-<component>-specs-<N>.md`

Assign each evaluator a distinct analytical approach:
- Evaluator 1: per-draft assessment (evaluate each draft individually, then rank)
- Evaluator 2: element-by-element deep dive (walk every design spec element,
  check each draft's coverage)
- Evaluator 3: topic-organized comparison (compare all drafts side-by-side on
  each dimension to identify best-of-breed per topic)

All evaluators run in parallel. They do not see each other's output.

## Phase 3 --- Convergence

After all evaluations complete, spawn one principal engineer to produce
convergence instructions.

The convergence agent receives:
1. All M evaluation reports (by path).
2. All N draft specs (for reference).
3. All design documents (for verification).
4. Instructions to:
   - Read all evaluation reports and identify consensus and disagreements
   - For each major topic, make a decision: which draft's approach to adopt,
     with rationale and specific instructions
   - Document any corrections or refinements identified during the session
     (the user may have provided feedback between phases)
   - Write to `reviews/converge-<component>-specs.md`
   - The convergence doc must be self-contained: the user should be able to
     produce the final spec from it alone

The convergence doc is organized by topic. Each topic has:
- **Decision**: what the spec should say
- **Source**: which draft(s) and evaluator(s) this comes from
- **Rationale**: why this choice wins
- **Instructions**: specific enough to write the spec section unambiguously

## Output

The orchestrator's deliverable is the convergence doc
(`reviews/converge-<component>-specs.md`). Writing the final `detailed-design.md` from
the convergence doc is a collaborative process between the user and the AI ---
it is not automated by this orchestrator. The spec is the implementation-level
design record; it persists across release cycles and evolves as
implementation-level decisions change.

## User Checkpoints

The orchestrator pauses for user review at these points:

- **After Phase 1**: user may review drafts, provide feedback, or skip to
  Phase 2.
- **After Phase 2**: user may review evaluations, provide corrections, or skip
  to Phase 3.
- **After Phase 3**: the orchestrator is done. The user works through the
  convergence doc topic by topic, correcting decisions, adding constraints,
  and resolving questions. The final spec is produced collaboratively from
  this refined convergence doc.

## Iteration

Draft specs use `draft-<N>.md` naming (1-indexed). When re-running, existing
drafts are overwritten. Evaluation reports and convergence docs are similarly
overwritten --- they are working documents, not archives.

## Cleanup

After the final spec is accepted, the following artifacts may be removed:
- `implementation/<component>/draft-*.md` --- working drafts
- `reviews/pe-<component>-specs-*.md` --- evaluation reports
- `reviews/converge-<component>-specs.md` --- convergence instructions

These are intermediate artifacts. The spec is the deliverable.

## Partitioning Strategy

Apply [agent-partitioning-strategy.md](agent-partitioning-strategy.md).
Phases 1 and 2 are embarrassingly parallel. Phase 3 is sequential and
single-agent.

## Orchestration Notes

- Phase 1 agents do not see each other's output.
- Phase 2 evaluators do not see each other's output.
- Phase 3 sees all evaluations but not Phase 1 drafts directly (only via
  evaluator citations). The convergence agent may reference drafts for
  detail when evaluators disagree.
