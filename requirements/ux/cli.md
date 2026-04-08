# CLI Output

| Path | Tag | Requirement |
|------|-----|-------------|
| R:ux / cli / projection-table | MVP | The plan must produce a year-by-year projection table via command-line invocation. |
| R:ux / cli / projection-table : columns | MVP | The projection table must include for each year in [D:streams / timeline](../conceptual-model.md#d-streams): calendar year, ages of all [S:household / [ member ]](../conceptual-model.md#s-household), balances of all [S:assets / accounts](../conceptual-model.md#s-assets), [S:income](../conceptual-model.md#s-income) by source, total [S:expenses](../conceptual-model.md#s-expenses), federal tax, and surplus or deficit. |
| R:ux / cli / projection-table : denomination | MVP | All monetary values in the projection table must be displayed per [D:denomination / display](../conceptual-model.md#d-denomination). |
| R:ux / cli / projection-table : missing-assumptions | MVP | The projection must not run if required [S:assumptions](../conceptual-model.md#s-assumptions) are missing. |
| R:ux / cli / projection-table : allocation-sum | MVP | The projection must not run if [S:assumptions / allocations](../conceptual-model.md#s-assumptions) do not sum to 100%. |
| R:ux / cli / output-format : json | MVP | The CLI must support JSON-formatted output for use by test infrastructure. |
