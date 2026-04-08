# Expenses

| Path | Tag | Requirement |
|------|-----|-------------|
| R:engine / expenses / living | MVP | The plan must support [S:expenses / living](../conceptual-model.md#s-expenses) as a recurring expense. |
| R:engine / expenses / living : inflation | MVP | [S:expenses / living](../conceptual-model.md#s-expenses) must inflate annually at [S:assumptions / inflation / cpi](../conceptual-model.md#s-assumptions). |
| R:engine / expenses / health / insurance / employer | MVP | The plan must support [S:expenses / health / insurance / employer](../conceptual-model.md#s-expenses) as a household expense. |
| R:engine / expenses / health / insurance / employer : phase | MVP | [S:expenses / health / insurance / employer](../conceptual-model.md#s-expenses) must be active only while at least one [S:household / [ member ]](../conceptual-model.md#s-household) has not reached [R:engine / timeline / [ member ] : retirement-year / phase-boundary](timeline.md). |
| R:engine / expenses / health / insurance / employer : inflation | MVP | [S:expenses / health / insurance / employer](../conceptual-model.md#s-expenses) must inflate at [S:assumptions / inflation / health](../conceptual-model.md#s-assumptions). |
| R:engine / expenses / health / insurance / aca | MVP | The plan must support [S:expenses / health / insurance / aca](../conceptual-model.md#s-expenses) as a household expense. |
| R:engine / expenses / health / insurance / aca : phase | MVP | [S:expenses / health / insurance / aca](../conceptual-model.md#s-expenses) must be active only while at least one [S:household / [ member ]](../conceptual-model.md#s-household) has reached [R:engine / timeline / [ member ] : retirement-year / phase-boundary](timeline.md) but has not reached [S:assumptions / policy / medicare / eligibility-age](../conceptual-model.md#s-assumptions). |
| R:engine / expenses / health / insurance / aca : inflation | MVP | [S:expenses / health / insurance / aca](../conceptual-model.md#s-expenses) must inflate at [S:assumptions / inflation / health](../conceptual-model.md#s-assumptions). |
| R:engine / expenses / health / insurance / medicare | MVP | The plan must support [S:expenses / health / insurance / medicare](../conceptual-model.md#s-expenses) as a household expense. |
| R:engine / expenses / health / insurance / medicare : phase | MVP | [S:expenses / health / insurance / medicare](../conceptual-model.md#s-expenses) must be active while at least one [S:household / [ member ]](../conceptual-model.md#s-household) has reached [S:assumptions / policy / medicare / eligibility-age](../conceptual-model.md#s-assumptions). |
| R:engine / expenses / health / insurance / medicare : inflation | MVP | [S:expenses / health / insurance / medicare](../conceptual-model.md#s-expenses) must inflate at [S:assumptions / inflation / health](../conceptual-model.md#s-assumptions). |
| R:engine / expenses / health / out-of-pocket | MVP | The plan must support [S:expenses / health / out-of-pocket](../conceptual-model.md#s-expenses) as a recurring expense. |
| R:engine / expenses / health / out-of-pocket : inflation | MVP | [S:expenses / health / out-of-pocket](../conceptual-model.md#s-expenses) must inflate at [S:assumptions / inflation / health](../conceptual-model.md#s-assumptions). |
