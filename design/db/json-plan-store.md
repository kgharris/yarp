# JsonPlanStore Format

`JsonPlanStore` is the production persistence backend. Plan files are
human-readable JSON, directly editable, and git-diffable.

---

## Directory Layout

A plan store is a directory containing a single file:

```
<plan-dir>/
    plan.json
```

A directory is a valid plan store if and only if it contains `plan.json`.
Any other contents are ignored.

---

## plan.json

`plan.json` contains the complete plan — all entities, streams, stream points,
assumptions, and lifecycle events — serialized as a single JSON object.

The root object carries a schema version field:

```json
{
  "schema_version": 1,
  "plan": { ... }
}
```

`schema_version` is an integer identifying the format this file was written
in. The DB layer reads this field before deserializing `plan`. If the version
does not match the version this binary supports, `PlanStoreError::SchemaMismatch`
is returned before any further parsing occurs. Schema migration is out of scope.

---

## Write Strategy

Writes use a temp-file-then-rename strategy to prevent partial writes:

1. Write the serialized JSON to `plan.json.tmp` in the same directory.
2. Rename `plan.json.tmp` → `plan.json` atomically.

A crash mid-write leaves the prior `plan.json` intact.
