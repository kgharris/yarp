# Context Assembly Protocol

When spawning an agent for any workflow, the orchestrator assembles the agent's
context by following these rules. The assembled context is delivered inline in
the agent's prompt — agents do not fetch principles themselves.

## Inputs

The orchestrator determines three values before assembling context:

- **personality** — which personality file governs this agent (may be none for
  generic tasks like fix application)
- **phase** — which phase the work is in: `requirements`, `design`,
  `implementation`, or `testing` (may be none for cross-phase work)
- **subsystem** — which subsystem the agent focuses on: `engine`, `db`,
  `controller`, `ux` (may be none for Forest/Branch agents)

## Assembly Order

Include the following, in order. Skip any file that does not exist for the
given combination. The order matters — each layer builds on the previous.

### Always included

1. `bedrock-principles.md` — foundational principles (separation of concerns, MVC)
2. `ai/personalities/principles.md` — altitude model, scope discipline

### Phase-dependent (when phase is known)

3. `ai/workflows/context/<phase>.md` — what this phase is, what it excludes,
   artifact locations, altitude-specific reading guidance
4. `<phase>/principles.md` — phase-level principles (design principles,
   implementation principles, testing principles). Not all phases have one.

### Subsystem-dependent (when subsystem is known)

5. `<phase>/<subsystem>/principles.md` — subsystem-specific principles and
   MVC constraints. Only included when both phase and subsystem are known.

### Personality-dependent (when personality is assigned)

6. `ai/personalities/<personality>.md` — role, altitude, what to flag, what
   to skip
7. `ai/personalities/context/<personality>/<phase>.md` — phase-specific focus
   and questions for this personality. Only included when both personality and
   phase are known.

## Rules

- **Inline, don't reference.** The assembled principles are delivered as part
  of the agent's prompt. The agent should never need to fetch a principles file
  itself. If the orchestrator tells an agent "go read X," the protocol has been
  violated.

- **No partial assembly.** If a personality is assigned, all applicable layers
  must be included. An agent that receives its personality file but not the
  bedrock principles is operating without its foundation.

- **One phase context.** An agent receives the context file for exactly one
  phase, never multiple. The orchestrator selects the phase before spawning.

- **Subsystem follows personality.** The subsystem is typically determined by
  the personality (Engine Engineer → engine, UX Engineer → ux). Forest agents
  have no subsystem. Branch agents may span subsystems but still receive the
  primary subsystem principles for their domain.

## Example: Engine Engineer, Design Phase

Assembled context (in order):
1. `bedrock-principles.md`
2. `ai/personalities/principles.md`
3. `ai/workflows/context/design.md`
4. `design/principles.md`
5. `design/engine/principles.md`
6. `ai/personalities/engine-engineer.md`
7. `ai/personalities/context/engine-engineer/design.md`

## Example: Principal Engineer, Requirements Phase

Assembled context (in order):
1. `bedrock-principles.md`
2. `ai/personalities/principles.md`
3. `ai/workflows/context/requirements.md`
4. (no `requirements/principles.md` — doesn't exist)
5. (no subsystem — Forest agent)
6. `ai/personalities/principal-engineer.md`
7. `ai/personalities/context/principal-engineer/requirements.md`

## Example: Fix Application (no personality)

Assembled context (in order):
1. `bedrock-principles.md`
2. `ai/personalities/principles.md`
3. (phase may or may not apply depending on what's being fixed)
