# Denomination

| Path | Tag | Requirement |
|------|-----|-------------|
| R:engine / denomination | CORE | Every dollar amount in the plan must be denominated in exactly one of the four reference frames defined in [D:denomination](../conceptual-model.md#d-denomination). |
| R:engine / denomination / yzv | MVP | All configuration and assumption values persisted in a plan must be stored in [D:denomination / yzv](../conceptual-model.md#d-denomination). |
| R:engine / denomination / yzv : anchor | MVP | The plan anchor year must be fixed at plan creation and must not change. |
| R:engine / denomination / ynv | MVP | Every dollar value the engine produces as projection output for a given year must be expressed in [D:denomination / ynv](../conceptual-model.md#d-denomination) for that year. |
| R:engine / denomination / ynv : derivation | MVP | Year nominal values must always be derived by applying inflation factors to stored [D:denomination / yzv](../conceptual-model.md#d-denomination) values — never sourced from observed external data. |
| R:engine / denomination / cnv | FUT | The plan must support conversion of stored [D:denomination / yzv](../conceptual-model.md#d-denomination) values to [D:denomination / cnv](../conceptual-model.md#d-denomination) for user-facing display, using cumulative inflation from the anchor year to the current calendar year. |
| R:engine / denomination / pnv | FUT | Historical actuals recorded in the plan — year-end balances, IRS-published limits, and other observed data — must be stored in [D:denomination / pnv](../conceptual-model.md#d-denomination), denominated in the nominal dollars of their reference date. |
| R:engine / denomination / pnv : invariance | FUT | A value recorded as [D:denomination / pnv](../conceptual-model.md#d-denomination) must not be modified once committed. |
| R:engine / denomination / display | MVP | Dollar values presented to the user must default to [D:denomination / cnv](../conceptual-model.md#d-denomination). |
