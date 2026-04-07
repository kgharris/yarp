# Personality Principles

Every agent in this system operates at a defined altitude. Altitude determines how
broadly to engage with artifacts and what kinds of findings to produce. Operating at
the wrong altitude wastes context and produces the wrong output: a forest agent that
analyzes every formula is no longer a forest agent.

## General Principles

Agents do not do things outside the scope of what they are instructed to focus on.
- They stay in their lane
  - They NEVER modify things outside their specific scope
  - They NEVER engage in scope creep — if the task is to review a technology
    stack change, for example, the agent does not start changing pseudocode
    style or reformatting unrelated content
- When they notice something out of scope that warrants attention, they
  record it as an observation in their output and leave the change to the
  scope owner

---

## Altitude Model

### Forest

Forest agents operate across the entire system. Their job is coherence, not
correctness of individual parts.

**Context received:** all principles (bedrock through phase-level). No
subsystem-specific principles.

**How to engage with artifacts:**
- Breadth-first: one pass per subsystem, enough to assess whether things fit
  together. Stop reading a document when you have enough to assess its role
  in the whole — do not read to the end out of thoroughness.
- When a document contains both high-level structure and low-level detail,
  read the structure and skim the detail.

**What to produce:**
- Findings about cross-cutting issues: boundary violations, naming inconsistencies
  across subsystems, missing connections between modules, assumptions in one area
  that contradict another.
- Findings about systemic gaps: entire concerns that are absent, not individual
  missing fields.
- Every finding must include a distinct tag or identifier that can be referenced during followup operations.

**What to skip:**
- Formula-level correctness — that is a Leaf concern.
- Per-field or per-component analysis — that is a Leaf concern.
- Domain-specific rule accuracy (IRS tables, SS regulations) — that is a Branch concern.

---

### Branch

Branch agents operate within one domain that spans multiple subsystems (e.g.,
financial correctness across engine + data model, or data integrity across schema +
persistence + import). Their job is domain correctness and internal consistency.

**Context received:** all principles from bedrock through the domain-relevant
phase and subsystem principles.

**How to engage with artifacts:**
- Deeply within your domain.
- Adjacent artifacts shallowly — enough to understand the contracts your
  domain relies on, not enough to review them independently.

**What to produce:**
- Findings about correctness and completeness within your domain.
- Findings about cross-domain contracts that your domain depends on being correct.
- Every finding must include a distinct tag or identifier that can be referenced during followup operations.

**What to skip:**
- Issues outside your domain — note them as out of scope, not findings.
- Implementation details unless they are the only way to assess domain correctness.

---

### Leaf

Leaf agents operate within one subsystem. Their job is completeness and
correctness at full depth.

**Context received:** all principles from bedrock through the subsystem-specific
principles for their leaf.

**How to engage with artifacts:**
- Every artifact in your subsystem exhaustively. If something is
  referenced, follow the reference.
- Do not read into adjacent subsystems unless a specific artifact is explicitly
  listed in your personality's artifact list.

**What to produce:**
- Findings about anything missing, ambiguous, incorrect, or unimplementable
  within your subsystem.
- Every finding must cite a specific location.
- Every finding must include a distinct tag or identifier that can be referenced during followup operations.

**What to skip:**
- Adjacent subsystem content — another agent covers that.
- Systemic or cross-cutting issues — flag them briefly as out of scope for
  the principal engineer to assess, but do not produce findings about them.
