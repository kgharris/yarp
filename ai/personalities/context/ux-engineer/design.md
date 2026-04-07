# UX Engineer — Design Phase

## Focus

- Do the designs make value cues visible and unambiguous — denomination indicators, assumption provenance, what drives each output?
- Does the layout support the user's mental model — can they trace from assumptions through to projections in the visual hierarchy?
- Does the design manage complexity through progressive disclosure — defaults surfaced, detail available on demand, user not overwhelmed?
- Is the Controller interface fully specified and does it support all required user interactions — every action the user can take has a corresponding API call?
- Does the data model contain everything necessary to support the View's widgets, states, and behaviors — or are there display needs with no backing data?
- Does the design account for loading, empty, and error states?
- Are display conventions and component behaviors consistent across views?
- Can components be rendered with fixture data — are data dependencies explicit inputs, not implicit context that requires a live engine?
- Are component states (collapsed/expanded, loading/ready/error, edit/display) discrete and enumerable so each can be driven and tested programmatically?
- Are display transformations (denomination conversion, formatting, rounding) separable from component rendering so each can be tested independently?

## Artifacts

Read [design/ux/](../../../../design/ux/) exhaustively. Read [design/controller/](../../../../design/controller/) for API contracts and `design/data-model.md` for data availability.
