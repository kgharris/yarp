# Withdrawals

MVP scope is projection-only: the engine models contributions and asset growth
across the full timeline. Withdrawals and distributions — voluntary, mandatory
(RMD), and sequenced — are deferred to a future release. All requirements in
this file are FUT.

| Path | Tag | Requirement |
|------|-----|-------------|
| R:engine / withdrawals / [ member ] / [ 401k ] / standard : withdrawal | FUT | The plan must support voluntary withdrawals from [S:assets / accounts / [ member ] / [ 401k ]](../conceptual-model.md#s-assets). |
| R:engine / withdrawals / [ member ] / [ 401k ] / standard : tax-treatment | FUT | Voluntary withdrawals from [S:assets / accounts / [ member ] / [ 401k ]](../conceptual-model.md#s-assets) must be taxed as ordinary income. |
| R:engine / withdrawals / [ member ] / [ 401k ] / rmd : computation | FUT | The plan must compute required minimum distributions from [S:assets / accounts / [ member ] / [ 401k ]](../conceptual-model.md#s-assets) using [S:assumptions / policy / rmd / uniform-lifetime-table](../conceptual-model.md#s-assumptions) and [S:assumptions / policy / rmd / start-age](../conceptual-model.md#s-assumptions). |
| R:engine / withdrawals / [ member ] / [ 401k ] / rmd : application | FUT | The plan must apply computed required minimum distributions as withdrawals from [S:assets / accounts / [ member ] / [ 401k ]](../conceptual-model.md#s-assets). |
| R:engine / withdrawals / [ member ] / [ 401k ] / rmd : floor | FUT | The required minimum distribution is a floor on the total withdrawal from [S:assets / accounts / [ member ] / [ 401k ]](../conceptual-model.md#s-assets) in any year it applies. |
| R:engine / withdrawals / roth-conversion / [ member ] | FUT | The plan must support Roth conversions — a paired withdrawal from a traditional account and contribution to a Roth account for a specific member, with distinct tax treatment. |
| R:engine / withdrawals / [ member ] / [ roth-ira ] | FUT | The plan must support qualified withdrawals from [S:assets / accounts / [ member ] / [ roth-ira ]](../conceptual-model.md#s-assets). |
| R:engine / withdrawals / [ bank ] | FUT | The plan must support withdrawals from [S:assets / accounts / [ bank ]](../conceptual-model.md#s-assets). |
| R:engine / withdrawals : sequence | FUT | The plan must apply a configurable withdrawal priority sequence across [S:assets / accounts](../conceptual-model.md#s-assets). |
| R:engine / withdrawals : sequence-default | FUT | The default withdrawal priority sequence is [S:assets / accounts / [ bank ]](../conceptual-model.md#s-assets) first, then [S:assets / accounts / [ member ] / [ 401k ]](../conceptual-model.md#s-assets), then [S:assets / accounts / [ member ] / [ roth-ira ]](../conceptual-model.md#s-assets). |
