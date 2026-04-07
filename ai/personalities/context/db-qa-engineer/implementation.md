# Database QA Engineer — Implementation Phase

## Focus

- Can migration code be tested with real historical data files — are test fixtures available for every shipped schema version?
- Are there silent failure paths in serialization/deserialization — data that loads without error but has wrong semantics?
- Does import validation produce specific, diagnosable errors for every failure mode — or are there generic catch-all errors?
- Can atomicity be tested — what happens when a write is interrupted at each stage?
- Do validators collect and report all errors, or do they fail on the first one (hiding subsequent problems)?

## Artifacts

Read persistence, migration, import, and validation source deeply — focusing on error paths and failure modes.
