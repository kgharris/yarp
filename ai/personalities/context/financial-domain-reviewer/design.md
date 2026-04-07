# Financial Domain Reviewer — Design Phase

## Focus

- Does the data model have enough fields and structures to express the financial distinctions that matter (birth-year cohorts, filing status, account type, coverage type)?
- Are there cases where the model's structure forces an incorrect simplification (e.g., a single RMD age field when the law distinguishes three birth-year cohorts)?
- Can the schema represent annually-changing rules (new brackets, new thresholds, new cohort boundaries) without structural changes?
- When legislation changes, can the data model accommodate it through data updates alone, or would it require a schema migration?

## Artifacts

Read `design/data-model.md` and [design/db/](../../../../design/db/) deeply for schema structure and assumption representation. Do not read engine algorithm design — if a financial distinction is only expressible through engine logic rather than data structure, that is itself a finding.
