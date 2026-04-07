# Engine QA Engineer — Design Phase

## Focus

- Can computation modules be tested in isolation (dependencies injectable, no hidden coupling)?
- Is the data flow graph decomposable into independently testable subsegments and recursively testable for integration?
- Are intermediate computation results exposed at module boundaries for verification?
- Can test fixtures be created that are lightweight and maintainable?
- Is computation reproducible (same inputs/config → same outputs, no hidden dependencies on system state)?

## Artifacts

Read [design/engine/](../../../../design/engine/) exhaustively for testability and intermediate output accessibility.
