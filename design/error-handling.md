# Error Handling

This document defines the error format and type classification for errors
surfaced by the system. It declares the structure, not an exhaustive catalogue
of every message — specific error messages are left to the implementation.
Error handling is owned by the Model layer — callers (CLI, Controller, test
harnesses) display formatted error strings as-is. They do not interpret,
reformat, or construct error messages themselves.

---

## Error Format

All errors follow this format:

```
<severity>: <type>: <message>
```

- **severity** — `error` for all fatal conditions in MVP.
- **type** — a short kebab-case noun identifying the error class.
- **message** — a structured description defined per error type below.

---

## Error Types

| Type | When | Message format |
|---|---|---|
| `system-error` | Filesystem or OS-level failure | OS error message verbatim (e.g., `No such file or directory (os error 2)`, `Permission denied (os error 13)`, `File exists (os error 17)`) |
| `invalid-plan` | Plan structure or content fails validation | Structured description per violation (see below) |
| `schema-mismatch` | Plan schema version is incompatible | `schema version <found>; expected <expected>` |
| `invalid-argument` | Conflicting or invalid command-line arguments | Human-readable description of the conflict |

---

## `invalid-plan` Messages

`invalid-plan` errors carry one or more `PlanViolation`s. Each violation
produces one line of output.

| Violation | Message |
|---|---|
| Plan directory is not a valid store | `not a valid plan store: <path>` |
| No members | `plan has no members` |
| No accounts | `plan has no accounts` |
| Missing required assumption | `S:<assumption-path>` (one line per missing field) |
| Allocation sum error | `allocation sums to <N>% in year <Y>` (one line per violation) |
| Label contains comma | `<field> contains a comma: <value>` (e.g., `given_name contains a comma: O,Brien`) |

When multiple violations exist, each is emitted as a separate line with the
full `error: invalid-plan: <message>` prefix.

---

## ModelError Mapping

`ModelError` (defined in [design/engine/api.md](engine/api.md#modelerror)) is the
typed error returned by the Model facade. Each variant maps to an error type:

| `ModelError` variant | Error type |
|---|---|
| `NotFound(path)` | `system-error` |
| `InvalidStore(path)` | `invalid-plan` |
| `SchemaMismatch { found, expected }` | `schema-mismatch` |
| `InvalidPlan(violations)` | `invalid-plan` |
| `InvalidArgument(message)` | `invalid-argument` |
| `Io(source)` | `system-error` |

---

## Caller Contract

Callers receive `ModelError` from the Model facade and render it using the
format above. The Model layer implements `Display` for `ModelError`, producing
the `<severity>: <type>: <message>` string. Callers write this string to
stderr and set the appropriate exit code. They do not:

- Parse or branch on the formatted string
- Add prefixes, suffixes, or additional formatting
- Define new error types or messages
