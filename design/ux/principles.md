# UX — Design Principles

The UX is the **View** layer.

## MVC Constraints

- The UX renders data it receives from the Controller. It does not fetch or transform raw
  model data directly.
- Zero financial computation happens in the View. If a value needs to be derived, it is
  derived by the Engine and delivered by the Controller.
- Display preferences owned by the View (dollar-basis toggle, active scenario selection,
  navigation state, loading states) are UI state, not financial state.
- The View does not decide *what* to show — it decides *how* to show what it is given.

---

## Consistency

The UX must achieve consistent look and feel through a centralized style system (CSS and
equivalent technologies). All visual properties — colors, spacing, typography, sizing — are
defined once in a style specification and referenced by name everywhere else.

**Antipattern:** specifying pixel sizes, hex color values, or other concrete visual values
inside view- or component-specific design documents. Those documents must reference named
style elements ("the warning color", "the standard card padding"), not raw values.

Raw values live in exactly one place: the style specification. View and component specs
that embed raw values are duplicating what belongs there, and will drift.

---

## Component Reuse

A UI pattern is specified once and referenced by name everywhere else. If two views need
the same thing, they use the same component — they do not each describe it independently.

If a view needs a variation, that variation is expressed as a parameter or variant of the
existing component, not as a new ad-hoc spec. Divergence in specs produces divergence in
implementation.

**Antipattern:** describing a table, chart, or card layout in full detail inside a
view-specific document when a component spec for that pattern already exists or should exist.

---

## State Completeness

A component spec is not complete until every state is defined. For any interactive element:

- Default
- Hover / focus
- Active / selected
- Disabled
- Loading
- Error
- Empty

Specs that define only the happy-path default state are incomplete. Unspecified states will
be invented inconsistently at implementation time.

---

## Data Primacy

Every visual element must serve data communication. This is a financial planning tool —
the data is the UI. Decorative elements that do not convey information or direct attention
do not belong in the spec.

**Antipattern:** visual embellishments (gradients, shadows, decorative icons, animation)
added to make a view feel polished rather than to communicate something specific. If a
visual element cannot be justified by what it communicates to the user, remove it from
the spec.

---

## Behavior Before Appearance

Specs must define what the UI *does* before defining how it looks. For each view or
component, establish the interaction model — what triggers what, what updates in response
to what — before specifying visual treatment.

**Antipattern:** a spec that describes a fully styled layout but leaves input handling,
update triggers, and error flows vague or absent. Appearance without behavior is a mockup,
not a design.

---

## Accessibility

Accessibility constraints apply to every component and are defined alongside the component,
not added afterward.

- Color must never be the sole carrier of meaning. Any state communicated by color must
  also be communicated by shape, label, or icon.
- All interactive elements must be keyboard-navigable.
- Contrast ratios must meet WCAG AA at minimum.

**Antipattern:** a component spec that defines visual states (error = red border) without
specifying the accessible equivalent (error = red border + error icon + aria-invalid).

---

## Layout by Structure, Not by Dimension

Layouts are defined by grid structure and the relationships between components — not by
fixed pixel coordinates or hardcoded widths. A layout spec should describe how components
relate to each other and to the grid; concrete dimensions are resolved by the style system
and the grid definition.

**Antipattern:** specifying that a sidebar is "280px wide" in a view-specific doc. The
sidebar width is a style system value. The view spec says "sidebar + main content area"
and the grid defines the proportions.
