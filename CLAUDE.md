# CLAUDE.md

## Principles

All work in this project is governed by the principles chain. These files are
included here so they are always active during interactive sessions. Orchestrated
workflows deliver them via the context assembly protocol instead.

@bedrock-principles.md
@requirements/conceptual-model.md
@requirements/principles.md
@design/principles.md
@ai/personalities/principles.md
@ai/workflows/context-assembly.md

## Markdown Formatting

When writing markdown files, do NOT add manual section numbers to headings.
Use `#` hash depth alone to express hierarchy. Never write `## 1. Foo` or
`### 2.3 Bar` — write `## Foo` and `### Bar`.

When referencing another file from a markdown document, ALWAYS use a markdown
link. Never write a bare path like `requirements/engine/data-persistence.md` —
always write `[data-persistence.md](requirements/engine/data-persistence.md)` or
equivalent descriptive link text.

## Review Commands

When asked to "run a review" or "run a full review", execute under the
[review-orchestrator.md](ai/workflows/review-orchestrator.md) workflow.

Derive parameters from natural language:

| Phrase                          | phase          | scope  |
|---------------------------------|----------------|--------|
| "full requirements review"      | requirements   | full   |
| "requirements review"           | requirements   | (diff) |
| "full design review"            | design         | full   |
| "design review"                 | design         | (diff) |
| "full implementation review"    | implementation | full   |
| "implementation review"         | implementation | (diff) |
| "full testing review"           | testing        | full   |
| "testing review"                | testing        | (diff) |

If the phrase doesn't clearly specify a phase, ask before proceeding.
"Full" → `scope=full`. Otherwise scope is omitted (defaults to git diff).

## Swarm Commands

When asked to "run a swarm" or "sweep", execute under the
[swarm-orchestrator.md](ai/workflows/swarm-orchestrator.md) workflow.

The swarm definition defaults to `reviews/swarm.md` (produced by the
"draft a swarm" command). The user may specify an alternative path explicitly.

- **scope override** (optional): if the user specifies a directory or glob
  (e.g., "scope only design/"), pass it as the scope override.

If `reviews/swarm.md` does not exist and no alternative path is given, prompt
the user to draft one first.

## Fix Commands

When asked to "run fixes" or "apply fixes", execute under the
[fix-orchestrator.md](ai/workflows/fix-orchestrator.md) workflow.

Derive parameters from natural language:

- **fix file**: if the user names a file, resolve it against `reviews/`
  when no directory prefix is given (e.g., "run fixes from fixes-design-1"
  → `reviews/fixes-design-1.md`). If no file is specified, scan `reviews/`
  for files matching `fixes-*-*.md`, pick the highest iteration number,
  and use that. Only one fix cycle is active at a time.
- **scope** (optional): if the user specifies a directory or glob
  (e.g., "to design/ only"), pass it as the scope filter.

## Implementation Commands

When asked to "implement" or "build from plan", execute under the
[implementation-orchestrator.md](ai/workflows/implementation-orchestrator.md)
workflow.

- **implementation plan**: the path to a plan file. If the user names it
  without a directory prefix, look in `.impl-plans/` (e.g., "implement
  engine-timeline" → `.impl-plans/engine-timeline/plan.md`).
- **scope** (optional): limit to a subset of files or components within
  the plan.

If no plan is specified and no path can be inferred, ask before proceeding.

## Draft Swarm Command

When asked to "draft a swarm" or "create a swarm definition", write a new
swarm definition file to `reviews/swarm.md` following the structure in
[sample-swarm.md](ai/workflows/sample-swarm.md). The user's prompt describes
the goals of the swarm — use it to populate the Objective, Scope, Search
Criteria, Context for Agents, and Output Filename sections.

After writing the draft, stop and present a summary so the user can review
before running it. Do not auto-launch the swarm.
