# Database QA Engineer — Requirements Phase

## Focus

- Are persistence requirements stated with enough precision to write acceptance tests — what constitutes "data preserved correctly"?
- Are there data durability scenarios the user expects to survive — application crash, version upgrade, platform migration — and can each be verified?
- Are validation requirements specific enough to construct test inputs — what exactly makes data "bad" and what should happen when it is?
- Are import/export requirements verifiable — can round-trip fidelity be defined and tested?

## Artifacts

Read [requirements/engine/](../../../../requirements/engine/) deeply for persistence, import, and validation requirements — focusing on testability and verifiability.
