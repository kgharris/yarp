# AI Assistance Architecture

This project uses a structured multi-agent system for reviews, fixes, and
implementation. The architecture separates concerns into three layers:
principles, personalities, and workflows.

## Principles Chain

Rules are stated once, in a hierarchy. Each layer builds on the one above:

```
bedrock-principles.md                  # foundational: MVC, streams, dollar denomination
  requirements/principles.md           # requirements-phase rules
  design/principles.md                 # design-phase rules
    design/{engine,db,controller,ux}/principles.md
  implementation/principles.md         # implementation-phase rules
    implementation/engine/principles.md
  tests/principles.md                  # testing-phase rules
  ai/personalities/principles.md       # altitude model, scope discipline
```

No file restates content from a file above it. Reference by link, never by
duplication.

## Personalities

Each agent has a personality file (`ai/personalities/<name>.md`) that defines:
role, altitude, what to flag, what not to flag, and communication style.

**Altitude** controls how broadly an agent engages:

| Altitude | Scope | Example |
|----------|-------|---------|
| **Forest** | Entire system, breadth-first | Principal Engineer |
| **Branch** | One domain across subsystems | Financial Domain Reviewer, DB Engineer |
| **Leaf** | One subsystem, exhaustive depth | Engine Engineer, UX QA Engineer |

Personalities: product-owner, principal-engineer, engine-engineer,
ux-engineer, controller-engineer, db-engineer, financial-domain-reviewer,
engine-qa-engineer, ux-qa-engineer, controller-qa-engineer, db-qa-engineer.

## Context Assembly

Orchestrators build each agent's context inline using the
[context assembly protocol](ai/workflows/context-assembly.md). Agents never
fetch principles themselves. The assembled context follows a fixed order:

1. Bedrock principles
2. Personality principles (altitude model)
3. Phase context (`ai/workflows/context/<phase>.md`)
4. Phase-level principles
5. Subsystem principles (when applicable)
6. Personality file
7. Phase-specific personality context

## Workflows

Orchestrators in `ai/workflows/` coordinate multi-agent operations:

- **[review-orchestrator.md](ai/workflows/review-orchestrator.md)** -- runs
  review agents across a phase (requirements, design, implementation, testing)
- **[swarm-orchestrator.md](ai/workflows/swarm-orchestrator.md)** -- runs a
  swarm of reviewers from a swarm definition file
- **[fix-orchestrator.md](ai/workflows/fix-orchestrator.md)** -- applies fixes
  from a review findings file
- **[implementation-orchestrator.md](ai/workflows/implementation-orchestrator.md)**
  -- builds from an implementation plan

These are invoked via natural-language commands defined in
[CLAUDE.md](CLAUDE.md).
