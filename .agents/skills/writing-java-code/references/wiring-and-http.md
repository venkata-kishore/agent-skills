# Wiring, HTTP and data access

## Composition root

- One composition root builds the object graph by hand, as layered wiring records (for example config, then infrastructure, then repositories, then services, then routes). Each object goes in the first layer that has all its dependencies.
- No DI container. Constructors take their dependencies; nothing reaches for a global.

## HTTP

- A lightweight HTTP server, not a full framework, unless the project already uses one.
- Routes are data: each registrar returns a list of route specs (method, path, required permission, handler), which is what the route-permission test walks (`building-backends`).
- Jackson for JSON. Bigint ids are written as strings. PATCH bodies use a three-state field wrapper type (`building-backends`).

## Data access

- jOOQ with the generated code committed. When debugging, the generated code is ground truth; fix codegen errors first.
- Repositories hold every SQL statement. Services receive only the DSL context from the transaction gateway, never a connection or a data source.
- Migrations are hand-written and forward-only, with a golden schema file and a drift check (`writing-database-code`).

## Logging

- slf4j is the only logging API in project code, with logback behind it.
- Bridge JUL and other logging APIs to slf4j, and install the bridge first in `main()`, before anything logs.
- What a log line may carry: `securing-applications`.
