# Project layout, CLI, server, database, build

These are defaults for a new project. An existing repo's choices win. Every library named here is an option, not a license to add it: each new dependency needs the user's approval and a check that the standard library does not already cover it. Use the latest stable version, verified against the registry and pinned.

## Module layout

- One `go.mod` at the repo root. Module path is the repo path.
- `cmd/<binary>/main.go` for each binary. `main` only parses flags and env, builds dependencies, calls `run`, handles exit. Nothing else.
- `internal/` for all application code. The compiler blocks outside imports, so you can refactor freely.
- No `pkg/`. The official layout guide never mentions it and most of the ecosystem does not use it. If something must be importable by other modules, split it into its own module when that consumer actually exists.
- Non-Go directories at the root: `web/`, `docs/`, `deploy/`, `scripts/`, `skills/`.
- Never commit `go.work`. It solves nothing for a Node frontend next to a Go module, shadows a developer's own workspace and makes CI test the wrong versions. If a second Go module appears, keep `go.work` gitignored.
- Never commit a local-path `replace`.

Package design:

- Group by domain noun (`internal/order`, `internal/invoice`), each owning its types, logic and the small store interface it needs. Adapters (HTTP, Postgres) either inside the domain package when small or in dependency-named packages (`internal/server`, `internal/store`). Pick one and be consistent.
- Dependencies point inward: server and store depend on domain packages, never the reverse.
- When packages of one layer import each other, or an import cycle appears, define the interface in the consumer or move the shared piece into a helper package both import, named for a real noun (never `types` or `common`). Enforce the direction with a guard (depguard, an import-rule Go test).
- Small sharp packages (`internal/reqctx`, `internal/build`, `internal/config`) instead of one `util`.

Reference tree for a CLI plus server plus web UI:

```
go.mod  Taskfile.yml  .goreleaser.yaml  Dockerfile  AGENTS.md
cmd/
  <app>/main.go           CLI
  <app>-server/main.go    API server (or a `serve` subcommand of the CLI)
internal/
  cli/                    cobra commands, one package per noun; cmdutil/ for Factory and JSON exporter
  client/                 Go HTTP client used by the CLI
  apitypes/               wire DTOs shared by client and server; JSON tags, no logic
  server/                 mux, middleware, route registration, graceful shutdown
  <domain>/               one package per noun with service logic
  store/
    migrations/           goose SQL, embedded
    queries/              sqlc input SQL
    sqlcgen/              sqlc output, committed
    pg/                   hand-written pgx for dynamic and bulk queries, tx helper
  config/  reqctx/  logging/  build/
web/                      Vue app; embed.go and noui.go; dist/.gitkeep committed
```

## CLI

Default: cobra with pflag, no viper. This is the GitHub CLI pattern.

One parser per flag kind: every command that takes the flag uses it (see `structuring-backend-code`).

Per command:

```go
type ListOptions struct {
	IO       *iostreams.IOStreams
	Client   func() (*client.Client, error)
	Exporter cmdutil.Exporter
	Limit    int
}

func NewCmdList(f *cmdutil.Factory, runF func(*ListOptions) error) *cobra.Command {
	opts := &ListOptions{IO: f.IOStreams, Client: f.Client}
	cmd := &cobra.Command{
		Use:   "list",
		Short: "List orders in the current project",
		RunE: func(cmd *cobra.Command, args []string) error {
			if runF != nil {
				return runF(opts)
			}
			return listRun(cmd.Context(), opts)
		},
	}
	cmd.Flags().IntVarP(&opts.Limit, "limit", "L", 10, "Maximum number of orders")
	cmdutil.AddJSONFlags(cmd, &opts.Exporter, orderFields)
	return cmd
}
```

- Options struct carries dependencies from a `Factory`. `runF` is the test seam. Expensive lookups happen inside `RunE`, not at construction.
- Every list and get command supports `--json` with a stable, tested field set. One shared exporter. Check `opts.Exporter != nil` before any human output.
- Stdout is data, stderr is diagnostics. Non-TTY output has no colour, no truncation, no prompts. Honour `NO_COLOR`.
- Errors teach: say what was wrong and the command that fixes it. Exit codes are stable.
- Help text and `--help` are read by models. Keep them exact and example-led.
- Config resolution: flag, then env (`APP_TOKEN`, `APP_API_URL`), then a small file under `$XDG_CONFIG_HOME/<app>/`, then defaults. Implement explicitly in about fifty lines. Tokens in the OS keyring with a 0600 file fallback.
- Version from ldflags with `debug.ReadBuildInfo()` fallback so `go install` still reports something.
- Shell completions and man pages generated at release time as a GoReleaser `before` hook.

Alternatives if the repo already uses them: kong (struct-tag CLIs, used by gum), urfave/cli v3. Declarative YAML-generated commands (Temporal CLI) are worth it once the command tree is large.

## HTTP server

Default: stdlib `net/http` with 1.22+ patterns. Add chi only if sub-router mounting becomes painful. Avoid gin and fiber (own context types; fiber is not net/http).

- Routes: `mux.HandleFunc("GET /api/orders/{id}", h.getOrder)`. `r.PathValue("id")`. `{$}` anchors an exact match. Conflicting patterns panic at registration. 405 with `Allow` is automatic.
- Middleware is `func(http.Handler) http.Handler`. Compose with a tiny chain helper. Needed set: request ID, access log via slog, body size limit, timeout, CORS for the dashboard, auth, client IP from trusted proxies only.
- Handlers return `error`; one adapter maps to status and logs once (see `errors.md`).
- Request-scoped values (request ID, principal, tenant) via one `reqctx` package with typed accessors. Everything else injected through constructors.
- Graceful shutdown:

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
srv := &http.Server{Addr: cfg.Addr, Handler: h, ReadHeaderTimeout: 10 * time.Second,
	BaseContext: func(net.Listener) context.Context { return ctx }}
errc := make(chan error, 1)
go func() { errc <- srv.ListenAndServe() }()
select {
case err := <-errc:
	return err // ListenAndServe always returns non-nil
case <-ctx.Done():
}
shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()
if err := srv.Shutdown(shutdownCtx); err != nil {
	return fmt.Errorf("shutdown: %w", err)
}
pool.Close()
return nil
```

`http.ErrServerClosed` after `Shutdown` is success, not failure.

- OpenAPI: huma v2 with the `humago` adapter if you want a code-first 3.1 spec and a generated TypeScript client. oapi-codegen strict-server if the spec file is the source of truth. Decide early; it determines where DTOs live.
- `/healthz` checks the database.

## Postgres

Default: pgx v5 via `pgxpool` plus sqlc with `sql_package: "pgx/v5"`. No ORM.

- Queries live in `.sql` files with sqlc annotations. Generated code is committed. `task generate` regenerates; CI fails on a dirty diff.
- Hand-written pgx (`pgx.CollectRows`, `pgx.NamedArgs`, `CopyFrom`, `LISTEN/NOTIFY`) in the same store package for dynamic filters and bulk work.
- sqlc structs never leak into API DTOs. Map in the store or service layer.
- Translate `pgx.ErrNoRows` to a domain `ErrNotFound` inside the store. Callers never import pgx.
- Pass `ctx` to every query. Use a transaction helper `func (s *Store) tx(ctx, func(q *sqlcgen.Queries) error) error` that commits or rolls back.
- Store tests against a real Postgres are integration tests: ask the user first, then build-tag them and use transaction-per-test for isolation.
- Schema rules (status values, no CHECK, DEFAULT or triggers, unique-index approval, bigint ids, UTC) are in `writing-database-code`.

Migrations: goose with embedded SQL (`//go:embed migrations/*.sql`, `goose.SetBaseFS`). One file per migration with `-- +goose Up` and `-- +goose Down`. Zero-pad versions (`00001_init.sql`) because sqlc reads them lexicographically. Expose `<app>-server migrate up` so self-hosters need no extra tool. In SaaS deploys run migrations as an explicit step, not on every instance boot. Alternatives already in use: tern (pgx-native), golang-migrate (needs paired up/down files).

## Configuration

- Server: environment variables only, parsed once into a typed struct (go-envconfig or caarlos0/env), validated in `Load()`, failing fast. Passed by value. No globals. Tests use a map lookuper so they stay parallel.
- Secrets via env or `<NAME>_FILE` path variants. Never flags; they show in `ps`.
- Same binary for self-host and SaaS, differing only by env. Build tags for compile-time exclusions only (UI, cloud-only modules).
- Avoid viper: case-insensitive keys, not goroutine-safe, implicit merging. koanf if layered file config becomes necessary.

## Embedding a web UI

```go
// web/embed.go
//go:build !noui

package web

import ("embed"; "io/fs")

//go:embed all:dist
var distFS embed.FS

const Embedded = true

func FS() (fs.FS, error) { return fs.Sub(distFS, "dist") }
```

```go
// web/noui.go
//go:build noui

package web

import "io/fs"

const Embedded = false

func FS() (fs.FS, error) { return nil, nil }
```

- Commit `web/dist/.gitkeep` and gitignore the rest of `dist/`. Without `all:` the pattern matches nothing when only a dotfile exists, and a missing `dist/` breaks `go build`, `go vet`, `go test` and lint, not just the server build.
- Build tags are per file, hence two complementary files.
- Default build embeds; `-tags noui` for API-only images and dev. In dev, proxy or CORS-allow the Vite dev server so HMR works.
- Vite `outDir` must resolve inside the Go package directory; `embed` cannot reach `..`.
- SPA fallback: unknown non-API, non-asset paths serve `index.html`. Keep `/api/` reserved.

## Build, release, containers

- Taskfile (`task --list` works for humans and agents). Targets: `generate`, `build`, `test`, `lint`, `dev`, `web:build`. Use `sources:`/`generates:` so unchanged UI builds are skipped. Logic that outgrows YAML moves to a Go program under `scripts/`.
- Pinned Go tools via `go get -tool ...` and `go tool <name>` (1.24+): sqlc, goose, oapi-codegen. Air is GPL-3.0 and would enter `go.mod`; use wgo (MIT) for Go live reload. Vite handles frontend HMR.
- `//go:generate go tool sqlc generate` style directives. `go generate` never runs implicitly; `task generate` calls it.
- GoReleaser: one build per binary with `ldflags: -s -w -X <module>/internal/build.Version={{.Version}}`, `CGO_ENABLED=0`. The UI build goes in the global `before.hooks`, not build-level hooks, which run once per OS and arch. `brews` is deprecated since v2.10; use `homebrew_casks`, which wants signing or a quarantine workaround. godownloader is archived; hand-write `install.sh` that verifies `checksums.txt`. Also publish deb/rpm via nfpm.
- Dockerfile: Node stage emits `web/dist`, Go stage builds with `CGO_ENABLED=0 -trimpath -ldflags "-s -w"`, final stage `gcr.io/distroless/static-debian13:nonroot`. JSON-array `ENTRYPOINT`. Pin the Debian suffix.

## Repo documents for agents

- A short `AGENTS.md` at the root: exact commands for build, lint, a single test, generate; the layout rules; the test scope. Route to topic docs rather than inlining everything.
- Conventions enforceable by linters or Go tests (import rules, license headers, generated-code drift) beat prose.
