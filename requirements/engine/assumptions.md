# Assumptions

| Path | Tag | Requirement |
|------|-----|-------------|
| R:engine / assumptions / inflation / cpi | MVP | The plan must support a configurable general inflation rate per [S:assumptions / inflation / cpi](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / inflation / health | FUT | The plan must support a healthcare inflation rate per [S:assumptions / inflation / health](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / inflation / cpi-w | FUT | The plan must support a CPI-W rate per [S:assumptions / inflation / cpi-w](../conceptual-model.md#s-assumptions) used for Social Security COLA adjustments. |
| R:engine / assumptions / rates / large-cap | MVP | The plan must support a configurable return rate assumption per [S:assumptions / rates / large-cap](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / rates / bonds | MVP | The plan must support a configurable return rate assumption per [S:assumptions / rates / bonds](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / rates / vehicle | MVP | The plan must support a configurable vehicle depreciation rate per [S:assumptions / rates / vehicle](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / allocations | MVP | The plan must support a configurable asset allocation per [S:assumptions / allocations](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / allocations : sum | CORE | [S:assumptions / allocations](../conceptual-model.md#s-assumptions) must sum to 100%. |
| R:engine / assumptions / policy / limits / 401k | FUT | The plan must enforce 401k contribution limits using [S:assumptions / policy / limits / 401k](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / limits / ira | FUT | The plan must enforce the combined IRA annual contribution limit using [S:assumptions / policy / limits / ira](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / limits / roth-ira / phase-out | FUT | Roth IRA direct contributions must phase out based on MAGI using [S:assumptions / policy / limits / roth-ira / phase-out](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / rmd / start-age | FUT | The RMD start age must be determined per a configurable birth-year-to-start-age mapping table per [S:assumptions / policy / rmd / start-age](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / rmd / uniform-lifetime-table | FUT | RMD calculations must use the distribution period divisor from [S:assumptions / policy / rmd / uniform-lifetime-table](../conceptual-model.md#s-assumptions) for the member's age in each projection year. |
| R:engine / assumptions / policy / ss / fra | FUT | Each income-earning [S:household / [ member ]](../conceptual-model.md#s-household)'s Full Retirement Age must be derived from their birth year using [S:assumptions / policy / ss / fra](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / ss / delayed-credit-rate | FUT | Social Security benefits claimed after FRA must increase using [S:assumptions / policy / ss / delayed-credit-rate](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / ss / taxation-thresholds | FUT | The taxable portion of Social Security benefits must be determined using [S:assumptions / policy / ss / taxation-thresholds](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / medicare / eligibility-age | FUT | Medicare eligibility age must be configurable per [S:assumptions / policy / medicare / eligibility-age](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / tax / federal / brackets | FUT | The plan must support configurable federal income tax brackets per [S:assumptions / policy / tax / federal / brackets](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / tax / federal / brackets : per-year-override | FUT | Federal income tax brackets must support per-year override capability. |
| R:engine / assumptions / policy / tax / federal / standard-deduction | FUT | The plan must support a configurable standard deduction per [S:assumptions / policy / tax / federal / standard-deduction](../conceptual-model.md#s-assumptions). |
| R:engine / assumptions / policy / tax / federal / standard-deduction : inflation | FUT | The standard deduction must inflate annually at [S:assumptions / inflation / cpi](../conceptual-model.md#s-assumptions) unless overridden. |
| R:engine / assumptions : defaults | MVP | Every [S:assumptions](../conceptual-model.md#s-assumptions) stream referenced in these requirements must have a shipped default value sufficient to produce a complete projection without user configuration. |
| R:engine / assumptions : override | MVP | Any [S:assumptions](../conceptual-model.md#s-assumptions) stream referenced in these requirements must be overridable on a per-year basis. |
| R:engine / assumptions : carry-forward | MVP | Per-year overrides of [S:assumptions](../conceptual-model.md#s-assumptions) streams referenced in these requirements must use carry-forward semantics. |
