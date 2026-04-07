# UX Engineer — Implementation Phase

## Focus

- Are value cues rendered correctly and kept in sync when context changes (dollar mode switch, scenario change, assumption update)?
- Does complexity management work as designed — progressive disclosure, collapsible state, default population?
- Does the implementation route all user actions through the Controller API — no direct engine or data layer access from the View?
- Are components implemented with explicit data inputs that can be supplied by fixtures, not implicit context requiring a live engine?
- Are display transformations (denomination conversion, formatting, rounding) implemented as separable functions, not inlined into component rendering?
- Are triggers explicit (on blur vs. on change vs. on submit)?
- Are focus management, ARIA roles, and keyboard interactions implemented correctly?
- Are display-layer boundaries from the principles chain respected?

## Artifacts

Read frontend source exhaustively.
