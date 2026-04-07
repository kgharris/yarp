# Financial Domain Reviewer — Implementation Phase

## Focus

- Are the IRS table values, tax rates, thresholds, and age triggers in the configuration/data files correct per current law?
- Do the configuration files expose enough parameters to express the distinctions that matter (birth-year cohorts, filing status, account type, coverage type)?
- Are there financial rules that the assumptions engine cannot represent — rules that would require a code change instead of a data update when legislation changes?
- When a new tax year arrives, can the user update the data files to reflect new law without touching code?

## Artifacts

Read configuration files, assumption defaults, and IRS/SSA parameter data deeply. Do not read engine source — if a financial rule is only discoverable in code, that is itself a finding.
