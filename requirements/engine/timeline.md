# Timeline & Demographics

| Path | Tag | Requirement |
|------|-----|-------------|
| R:engine / timeline | CORE | The plan must span a defined range of projection years per [D:streams / timeline](../conceptual-model.md#d-streams). |
| R:engine / timeline / [ member ] | CORE | Each [S:household / [ member ]](../conceptual-model.md#s-household) must have a derivable age for every year in the projection range. |
| R:engine / timeline / [ member ] : retirement-year | MVP | Each working-phase [S:household / [ member ]](../conceptual-model.md#s-household) must have a configurable retirement year. |
| R:engine / timeline / [ member ] : retirement-year / phase-boundary | MVP | The retirement year must mark the boundary between working and retired phases for that member. |
| R:engine / timeline / [ member ] : ss-claiming-age | FUT | Each [S:household / [ member ]](../conceptual-model.md#s-household) who has Social Security benefits configured must have a configurable Social Security claiming age. Deferred: SS income is FUT; no MVP feature reads this value. |
| R:engine / timeline / [ member ] : ss-claiming-age / range-constraint | FUT | The Social Security claiming age must fall within the range defined by [S:assumptions / policy / ss / fra](../conceptual-model.md#s-assumptions). Deferred: depends on SS income (FUT) and [S:assumptions / policy / ss / fra](../conceptual-model.md#s-assumptions) policy stream, which has no MVP R: mapping. |
| R:engine / timeline / [ member ] : rmd-start-age | FUT | Each [S:household / [ member ]](../conceptual-model.md#s-household) who owns a tax-deferred account subject to RMD rules must have an RMD start age derived from their birth year using [S:assumptions / policy / rmd / start-age](../conceptual-model.md#s-assumptions). Deferred: depends on RMD withdrawals (FUT) and [S:assumptions / policy / rmd / start-age](../conceptual-model.md#s-assumptions) policy stream, which has no MVP R: mapping. |
| R:engine / timeline / [ member ] : dependent-age-out | FUT | Each dependent [S:household / [ member ]](../conceptual-model.md#s-household) must have a configurable age-out age. Deferred: depends on dependent relation (FUT). |
| R:engine / timeline / [ member ] : dependent-age-out / budget-exit | FUT | When a dependent reaches their age-out age, they must leave the household budget. Deferred: depends on dependent relation (FUT). |
