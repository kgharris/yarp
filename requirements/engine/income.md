# Income

| Path | Tag | Requirement |
|------|-----|-------------|
| R:engine / income / w2 / [ member ] / [ employer ] | MVP | The plan must support [S:income / w2 / [ member ] / [ employer ]](../conceptual-model.md#s-income) for each income-earning [S:household / [ member ]](../conceptual-model.md#s-household), active through that member's retirement year. |
| R:engine / income / ss / [ member ] : pia | MVP | The plan must compute [S:income / ss / [ member ]](../conceptual-model.md#s-income) from each member's PIA. |
| R:engine / income / ss / [ member ] : claiming-adjustment | MVP | The plan must adjust [S:income / ss / [ member ]](../conceptual-model.md#s-income) based on claiming age relative to [S:assumptions / policy / ss / fra](../conceptual-model.md#s-assumptions). |
| R:engine / income / ss / [ member ] : cola | MVP | The plan must apply annual COLA to [S:income / ss / [ member ]](../conceptual-model.md#s-income) per [S:assumptions / inflation / cpi-w](../conceptual-model.md#s-assumptions). |
