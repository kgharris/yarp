# Requirements Context

Artifacts live under [requirements/](../../../requirements/).

## Structure

- `engine/` — computation and data requirements
- `ux/` — user interface and interaction requirements

## What Requirements Are

Requirements define the problem to be solved and the behaviors the system must exhibit.

- Externally observable behavior and outcomes
- Business rules and constraints
- User needs and acceptance criteria
- Technology-agnostic — multiple designs could satisfy them
- Testable without reference to implementation details

Examples:
- "The system must compute required minimum distributions according to IRS Uniform Lifetime Table III"
- "Users must be able to compare projected outcomes under different retirement ages"
- "All dollar amounts must be convertible between present-value and future-value representations"

## What Requirements Exclude

- **No data structures or field names** — `TimelineYear` with these fields is design
- **No algorithms or pseudocode** — `balance / divisor` is design
- **No specific file formats or schemas** — "store in JSON with field `rmd_start_age`" is design
- **No UI specifications** — pixel dimensions, component names, layout details are design
- **No libraries, frameworks, or language choices** — those are implementation

Test: could a domain expert validate this without knowing the tech stack? If not, it doesn't belong here.

## Altitude

- **Forest:** survey `engine/` and `ux/` at breadth
- **Branch:** read your domain's subtree deeply
- **Leaf:** read your subsystem's subtree exhaustively
