# Engine — Design Principles

The engine is the **Model** layer.

## MVC Constraints

- Engine modules are pure computation: given inputs, produce outputs.
- No module has any awareness of HTTP requests, API payloads, or response formats.
- No module knows how its output will be displayed or in what dollar basis.
- Modules do not call each other. The Controller orchestrates the pipeline.
- Interfaces are defined in terms of domain data structures, not serialization formats.
