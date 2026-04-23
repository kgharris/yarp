# CLI Output

| Path | Tag | Requirement |
|------|-----|-------------|
| R:ux / cli / projection-table | MVP | The plan must produce a year-by-year projection table via command-line invocation. |
| R:ux / cli / projection-table : columns | MVP | The projection table must include for each year in [D:streams / timeline](../conceptual-model.md#d-streams): calendar year, ages of all [S:household / [ member ]](../conceptual-model.md#s-household), and balances of all [S:assets / accounts](../conceptual-model.md#s-assets). |
| R:ux / cli / projection-table : denomination | MVP | All monetary values in the projection table must be output in YNV — denominated in each projection year's nominal dollars. |
| R:ux / cli / projection-table : missing-assumptions | MVP | The projection must not run if required [S:assumptions](../conceptual-model.md#s-assumptions) are missing. |
| R:ux / cli / projection-table : allocation-sum | MVP | The projection must not run if [S:assumptions / allocations](../conceptual-model.md#s-assumptions) do not sum to 100%. |
| R:ux / cli / output-format : json | MVP | The CLI must support JSON-formatted output for use by test infrastructure. |
| R:ux / cli / plan-input | MVP | The CLI must accept a path to a directory of JSON plan files as its plan data source. The files in that directory must be in the same format used by the DB layer. |
| R:ux / cli / generate-plan | MVP | The CLI must provide a `--generate-plan` option that writes a valid initial plan directory to a specified path. The generated files must be editable by the user to describe their specific plan. `--generate-plan` and projection mode are mutually exclusive — invoking both in the same command must be treated as an error. |
| R:ux / cli / generate-plan : runnable | MVP | The plan produced by `--generate-plan` must be immediately runnable — the CLI must be able to project it without error before the user makes any edits. |
| R:ux / cli / generate-plan : confirmation | MVP | On success, `--generate-plan` must emit exactly one line to stdout: `Plan created in <dir>`, where `<dir>` is the path supplied by the user. |
