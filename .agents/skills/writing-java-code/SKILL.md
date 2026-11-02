---
name: writing-java-code
description: "Sets Java and JVM conventions for backends: JDK version and language features (records, sealed types, pattern matching, enums), Gradle with Kotlin DSL, version catalogs, dependency locking and verification, multi-module layout, manual wiring without a DI framework, jOOQ, Jackson, slf4j and logback, exceptions and Optional, JUnit, AssertJ and Testcontainers. Use when writing, reviewing or structuring Java code, build.gradle.kts or libs.versions.toml, or a JVM service, test or build failure."
---

# Writing Java code

Backend rules for any language: `building-backends`. This skill adds the Java and JVM mechanics. Data layer: `writing-database-code`. Tests: `writing-tests` (JVM timeouts and containers: its `references/jvm.md`). Security: `securing-applications`.

## References

- `references/build.md`: Gradle layout, version catalog, locking and verification, version gates, the empty-home regeneration. Open when touching the build or any dependency.
- `references/wiring-and-http.md`: composition root, routes as data, jOOQ repositories, JSON, logging setup, error mapping. Open when adding a module, route, repository or error type.

## Before writing any Java

1. Check the toolchain version in the build. Use the latest LTS JDK (25 as of 2026-11). Do not use a feature newer than the toolchain.
2. Never guess a library API or version. Read the jar (`javap -cp <jar> <class>`) and the registry's own `maven-metadata.xml`, not a search page: search indexes lag.
3. Match the repo. Its build, wiring and test style win over this skill's defaults.

## Language

- Records for value types, request and response shapes, and carriers between repository and service. A record used by one class sits beside it; a record crossing a module boundary goes in that module's models package.
- Sealed interfaces with an exhaustive `switch` for every closed set. The point is that a new case breaks every caller at compile time, so no `default` branch.
- Enums carry an explicit stable code (stored as `smallint`). Never persist `ordinal()`. Enums that appear in stored history are append-only.
- `var` only when the type is obvious on the same line.
- Plain blocking code. No reactive types. Do not build on virtual threads unless the user decides to.
- No reflection-based DI and no annotation processors that generate behaviour. No Lombok.
- Set the JVM default time zone to UTC in a static initialiser that is the first member of the main class, before any logger or other static field initialises.
- Split a class over about 400 lines by responsibility. A split moves code; it does not rewrite it.

## Errors and nulls

- Exceptions, not result types. Client-facing refusals extend one base exception with a fixed message (never text built from input) and a kind that maps to a status and `errorCode`. One mapper builds the error envelope (`building-backends`).
- An unexpected `Throwable` becomes a logged, generic 500 with no exception text in the body. A framework HTTP exception keeps its own status (a 304 must not become a 500).
- Mail and transport exceptions are rewrapped with the class name only, never the original message or cause: those can quote addresses or payloads.
- `Optional` only as a return type for "may be absent". Never a field or parameter.
- Nullness: state the contract in javadoc where a caller needs it. No nullness annotation framework.
- Use the shared typed body reader and the named status methods, not local null checks or raw string comparisons.

## Layout

- Modules: core (domain, repositories, services), api (HTTP), workers (background jobs). Packages by layer first, then by entity.
- A guard script enforces what each layer may import and forbids import cycles between packages of one layer. A javadoc `{@link}` counts as an import: name the target in plain words when the link alone would close a cycle.

## Testing mechanics

- JUnit (latest major, 6.x as of 2026-11) with AssertJ. Hand-rolled fakes; no Mockito or other mocking library.
- Shared test code lives in a `testFixtures` source set (the `java-test-fixtures` plugin): database support, log capture, a mutable clock, race helpers.
- Testcontainers start once per JVM from a JUnit `LauncherSessionListener`, before any timed test. The same listener installs the logging bridge.
- A default per-test timeout in `junit-platform.properties`; a test that needs longer declares its own `@Timeout`.
- Inject a `Clock`. Truncate "now" where the stored precision is lower than the JVM's.

## Gates

- `./gradlew build` runs compile, tests, version gates and hygiene checks. Guard scripts run beside it (layering and cycles, database access only through the gateway, migration lint, emitted-constants drift, forbidden names). Each fails closed and also reads comments.
- javac stops at 100 errors per module. To compare a broken tree before and after a change, raise the limit with `-Xmaxerrs` from a scratch Gradle init script, never in the committed build.

## Common mistakes

- Guessing an API or version instead of reading the jar and the registry metadata.
- Copying a helper instead of reusing or extracting the shared one.
