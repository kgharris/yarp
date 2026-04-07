# Database QA Engineer — Testing Phase

## Focus

- Do migration tests use real historical data files, not just synthetic fixtures?
- Do tests cover every schema version transition path — including multi-step migrations (v1→v3)?
- Do round-trip tests verify that data survives save/load and import/export without semantic loss?
- Are corruption scenarios tested — malformed input, truncated files, wrong schema version, missing fields?
- Do tests verify that every failure is loud — no silent data loss, no generic error messages?
- Are validation rule tests isolated — each rule tested independently with both passing and failing inputs?

## Artifacts

Read data layer tests deeply. Verify test fixtures include real historical data files for every shipped schema version.
