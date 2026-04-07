# Database QA Engineer — Design Phase

## Focus

- Can migration paths be tested in isolation — is migration code separable from the application runtime?
- Can test fixtures be constructed for every schema version, including historical versions?
- Are there data states the schema can represent that would be invalid — and is validation designed to catch them?
- Is the validation lifecycle testable at each stage (load, edit, save, import)?
- Can atomicity guarantees be verified under test — interrupted writes, concurrent access?

## Artifacts

Read `design/data-model.md` and [design/db/](../../../../design/db/) deeply for testability of migration, validation, and persistence designs.
