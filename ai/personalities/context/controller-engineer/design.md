# Controller Engineer — Design Phase

## Focus

- Does the Controller design respect MVC boundaries (no business logic, no display logic)?
- Is the Model facade pattern properly designed (Controller → Model facade → DB)?
- Is the View presented with a clearly specified and complete Controller interface that supports all necessary user actions?
- Does the Model present a clearly specified and complete facade interface in support of the user actions required by the View?
- Are API contracts clean with well-defined request/response schemas?
- Is the Controller stateless (no session state, no cached data)?
- Are error handling and validation clearly assigned to the right layer?

## Artifacts

Read [design/controller/](../../../../design/controller/) exhaustively for API contracts and architectural boundaries.
