# Engine — Design Principles

The engine is the **Model** layer.

## MVC Constraints

- Engine modules are pure computation: given inputs, produce outputs.
- No module has any awareness of HTTP requests, API payloads, or response formats.
- Modules do not call each other. The Controller orchestrates the pipeline.
- Interfaces are defined in terms of domain data structures, not serialization formats.

## Denomination Conversion

The Engine is the sole site of denomination conversion. All four conversions
(YZV ↔ CNV, YZV → YNV, CNV → YNV, PNV ↔ YZV) are performed by the Engine.
No layer above the Engine converts dollar amounts.

The Engine accepts a target denomination as a parameter alongside the plan data.
The Controller forwards the denomination requested by the UX; it does not select
or apply a denomination itself.
