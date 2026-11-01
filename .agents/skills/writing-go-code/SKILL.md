---
name: writing-go-code
description: "Sets Go conventions for error handling, context, concurrency, generics, iterators, slog, testing, tooling, project layout, cobra CLIs, net/http servers, pgx + sqlc, migrations, embedded web UIs, Taskfile, GoReleaser and Dockerfiles. Use when writing, reviewing, or structuring Go code, or when go.mod, goroutines, slices and maps, or a Go project layout come up."
---

# Go Conventions

Checked against Go 1.27 (current stable). Where the style guides (Effective Go, Code Review Comments, Google, Uber) disagree, this skill picks a default and says so.

## Before writing any Go

1. Read `go.mod`. The `go` line gates language semantics and APIs. Do not use a feature newer than it. Table in `references/language.md`.
2. Match the repo. Existing lint config, test style (testify or stdlib), router, DB layer and task runner win over this skill's defaults.
3. Never emit `?`, `try()`, or `check`/`handle`. The Go team formally declined all error-handling syntax proposals in June 2025. Write explicit `if err != nil { return ..., err }`.
4. After generating code run `gofmt`, `go vet ./...`, and `go fix ./...` from a clean tree. `go fix` (1.26+) rewrites stale idioms that models keep emitting (`x := x`, manual min/max, `interface{}`, `b.N` loops).

## Core rules

Naming and shape:

- Packages: lowercase, one word, named for what they provide. Never `util`, `common`, `helpers`, `base`, `types`, `models`-as-dumping-ground. If no name works as a prefix for the contents, the package boundary is wrong.
- No stutter: `http.Server` not `http.HTTPServer`. No `Get` prefix on getters. Initialisms keep one case: `userID`, `ServeHTTP`, `parseURL`.
- Receivers: one or two letters, never `this`/`self`. Pointer receiver if the method mutates, the type holds a mutex, or the struct is large. Never mix kinds on one type.
- Make the zero value useful. Prefer `var s []string` over `[]string{}`. Check emptiness with `len(s) == 0`.
- Use `any`, not `interface{}`. Use `new(expr)` (1.26+) for pointers to values.
- Doc comment on every exported identifier, full sentence starting with the name.

Interfaces and dependencies:

- Define interfaces in the consuming package. Producers return concrete types. Do not pre-build interfaces for mocking.
- Keep interfaces at one or two methods. Compile-time guard when it matters: `var _ http.Handler = (*Server)(nil)`.
- Embed interfaces freely. Avoid embedding concrete types in exported structs (Uber position; chosen over Effective Go).
- No mutable package-level state. Inject loggers, clocks, config, DB handles through constructors. `init()` only if deterministic and I/O-free.
- `os.Exit` and `log.Fatal` only in `main`. Put logic in `func run(ctx, args, stdout) error` so defers run and it is testable.

Errors (details in `references/errors.md`):

- Return `error`, never a concrete pointer type. Never return a typed nil pointer as an error.
- Wrap with `fmt.Errorf("open config: %w", err)`. Context goes first, `%w` last, no "failed to" filler, no repeating what the inner error already says.
- Compare with `errors.Is`, extract with `errors.AsType[E]` (1.26+) or `errors.As`. Never compare error strings.
- Handle each error once: return it or log it, not both.
- Panic only for programmer errors and startup invariants. No blanket `recover`.

Context and concurrency (details in `references/concurrency.md`):

- `ctx context.Context` is the first parameter, named `ctx`. Never stored in a struct. `defer cancel()` right after creating one.
- Every goroutine has an owner, an exit path and a waiter. Use `wg.Go(f)` (1.25+) or `errgroup`. Every blocking send or receive pairs with `case <-ctx.Done()`.
- Loop variables are per-iteration when `go >= 1.22`. Do not write `v := v` in those modules.

Logging:

- `log/slog` for all new code. `logger.InfoContext(ctx, "msg", "key", val)`. Odd argument counts are bugs that vet catches.
- Pass `*slog.Logger` explicitly. slog deliberately rejected logger-in-context. Put request IDs in context and read them in a Handler.
- Log lines never carry secrets, tokens or raw identifiers. Errors that leave the server are generic (`securing-applications`).

Security, data and dependencies:

- `crypto/rand` for tokens and keys (`rand.Text()` on 1.24+). Never `math/rand`.
- `net.JoinHostPort`, never `fmt.Sprintf("%s:%d")`.
- `os.OpenRoot` (1.24+) for user-supplied paths.
- Copy slices and maps at API boundaries. Returning an internal slice leaks aliasing.
- Dependencies: latest stable, verified against the registry, pinned. Every new dependency needs the user's approval and a check that the standard library does not already cover it.

Sibling skills (follow them, do not restate): `structuring-backend-code` (layers, shared code, one owner per rule), `writing-tests`, `writing-database-code`, `securing-applications`.

## References

Open only the one the task touches.

- `references/errors.md`: wrapping, sentinels vs typed errors, HTTP handler error pattern, panics, CLI exit codes.
- `references/concurrency.md`: WaitGroup.Go, errgroup, pipelines, select, leaks, race detector, synctest.
- `references/language.md`: version-to-feature table, slices and maps, nil interfaces, shadowing, defer, generics, iterators, strings, time, slog, JSON.
- `references/testing.md`: test scope and quality bar, table tests, helpers, golden files, fuzzing, benchmarks, lint, pre-finish checklist.
- `references/layout.md`: module layout, cmd/internal, cobra CLI, net/http server, pgx + sqlc, goose, config, embedding a UI, Taskfile, GoReleaser, Dockerfile.

## Common mistakes

- `errors.As` with a non-pointer target, or `errors.Unwrap` on a joined error (it returns nil; use `Is`/`As`).
- `defer` inside a loop expecting per-iteration release. Wrap the body in a function.
- `:=` inside an `if` or `for` shadowing an outer `err` or `ctx`.
- `wg.Add(1)` inside the goroutine instead of before it. Prefer `wg.Go`.
- `context.Background()` inside a handler or deep in a call chain. Use the one you were given, or `t.Context()` in tests.
- `m[k].field = v` on a map of structs. Not addressable. Store pointers or reassign.
- `append` result not assigned, or two appends from one base slice overwriting each other.
- `time.After` in a hot loop. Use a reusable `time.NewTimer`.
- `pkg/` directory, `tools.go` blank-import file, committed `go.work`, local-path `replace`.
- Matching on JSON error text. `encoding/json` is backed by v2 since 1.27, so error text may differ.
