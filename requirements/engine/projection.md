# Projection

| Path | Tag | Requirement |
|------|-----|-------------|
| R:engine / projection / account-balance | CORE | Each [S:assets / accounts](../conceptual-model.md#s-assets) stream must produce a projected year-end balance for every year in [D:streams / timeline](../conceptual-model.md#d-streams). |
| R:engine / projection / account-balance / recurrence : end-of-year | MVP | [S:assets / accounts](../conceptual-model.md#s-assets) balances must be projected using [D:formulas / balance-recurrence / end-of-year](../conceptual-model.md#d-formulas). |
| R:engine / projection / account-balance / recurrence : beginning-of-year | FUT | [S:assets / accounts](../conceptual-model.md#s-assets) balances must support [D:formulas / balance-recurrence / beginning-of-year](../conceptual-model.md#d-formulas). |
| R:engine / projection / account-balance / recurrence : mid-year | FUT | [S:assets / accounts](../conceptual-model.md#s-assets) balances must support [D:formulas / balance-recurrence / mid-year](../conceptual-model.md#d-formulas). |
| R:engine / projection / account-balance : rate-derivation | MVP | The effective growth rate applied in [D:formulas / balance-recurrence](../conceptual-model.md#d-formulas) must be the weighted average of [S:assumptions / rates](../conceptual-model.md#s-assumptions) using [S:assumptions / allocations](../conceptual-model.md#s-assumptions) as weights. |
| R:engine / projection / account-balance : floor | CORE | No [S:assets / accounts](../conceptual-model.md#s-assets) balance may go below zero. |
| R:engine / projection / income-tax : taxable-income | MVP | The plan must compute federal taxable income for each year in [D:streams / timeline](../conceptual-model.md#d-streams) by applying [S:assumptions / policy / tax / federal / standard-deduction](../conceptual-model.md#s-assumptions). |
| R:engine / projection / income-tax : liability | MVP | The plan must compute federal income tax owed for each year in [D:streams / timeline](../conceptual-model.md#d-streams) by applying [S:assumptions / policy / tax / federal / brackets](../conceptual-model.md#s-assumptions) to taxable income. |
| R:engine / projection / ss-taxation | MVP | The taxable portion of [S:income / ss / [ member ]](../conceptual-model.md#s-income) must be determined each year using [S:assumptions / policy / ss / taxation-thresholds](../conceptual-model.md#s-assumptions). |
| R:engine / projection / surplus-deficit | MVP | Each year in [D:streams / timeline](../conceptual-model.md#d-streams) must produce an observable surplus or deficit equal to total [S:income](../conceptual-model.md#s-income) after tax minus total [S:expenses](../conceptual-model.md#s-expenses). |
