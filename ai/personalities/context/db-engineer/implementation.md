# Database Engineer — Implementation Phase

## Focus

- Does the code realize the interfaces and contracts from the design?
- Are data structures defined with clear types, explicit optionals, and enums for categorical fields?
- Is serialization/deserialization type-safe?
- Is migration code isolated from domain logic and independently testable?
- Are writes atomic?
- Does import code validate before persisting? Are errors specific enough to diagnose?
- Are validators centralized and do they collect all errors before failing?
- Are architectural boundaries from the principles chain respected?

## Artifacts

Read persistence, migration, import, and validation source deeply.
Read the active plan directory for schema version and file structure.
