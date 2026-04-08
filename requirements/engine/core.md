# Core

| Path | Tag | Requirement |
|------|-----|-------------|
| R:engine / core / precision | CORE | All internal monetary calculations must use arbitrary-precision decimal arithmetic. No intermediate or final engine result may lose precision due to fixed-width representation. |
| R:engine / core / rounding | CORE | Rounding is a presentation-layer concern. The engine must not apply rounding to any computed value. Rounding for display is the responsibility of the view layer. |
