# Engine Engineer — Design Phase

## Focus

- Does the Engine design respect MVC boundaries (pure computation, no Controller mediation logic, no display formatting)?
- Does the Engine present a clearly specified and complete computational interface to the Controller?
- Is the data model fully specified, maintainable, and understandable?
- Does the data model maintain proper separation of concerns without introducing unusable granularity?
- Are computation modules cleanly separated with explicit dependencies?
- Are module interfaces (inputs and outputs) explicitly typed and complete?
- Is the computation order unambiguous and enforced by design?
- Is the Engine stateless (no session state, computation from inputs only)?

## Artifacts

Read [design/engine/](../../../../design/engine/) exhaustively. Read `design/data-model.md` for data structures the engine consumes and produces.
