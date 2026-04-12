# Controller — Design Principles

The controller is the **mediating** layer between Model (Engine + DB) and View (UX).

## MVC Constraints

- The Controller owns the command surface: request validation, response shape.
- It orchestrates the Engine pipeline — deciding which modules run and in what order
  given a change to assumptions.
- It transforms raw engine output into view-ready payloads (resolving scenario
  overlays, serializing for the IPC bridge).
- It does not compute financial values, and it does not convert denominations.
  If a computation or conversion is needed, it calls the Engine.
- The UX communicates the desired denomination with each request; the Controller
  forwards it to the Engine as a parameter.
- It does not render anything. If a display decision is needed, it belongs in the UX.
- Scenario state (which scenario is active, overlay resolution) is a Controller responsibility.
- It calls the Model facade for persistence operations — it never interacts with the
  DB layer directly.
