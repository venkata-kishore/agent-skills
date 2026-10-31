# Testing and quality gates

## Scope and quality bar

- Default scope: unit tests for exported functions and methods, running with zero infrastructure. No live Postgres, Redis, network or filesystem outside `t.TempDir()`. Ask the owner before writing integration or end-to-end tests unless the project says otherwise. When allowed, put them behind a build tag (`//go:build integration`) or an env check that calls `t.Skip`. The `test-discipline` skill holds the shared rules.
- Every test must pin a behaviour that can regress: a transformation, a branch, a boundary, a contract with consequences. Do not write field-passthrough checks, "does not panic" checks, defaults the language already guarantees, or the same logic through trivially different inputs. Table tests are fine only when each row pins a distinct behaviour. Test count is not a quality signal. When a helper agent writes tests, state this rule in its prompt and prune violations.
- Fakes are hand-rolled. No mocking library (no `testify/mock`, `gomock`, `mockery`). Inject the clock; never read the wall clock. Test output stays pristine: no stray logs or prints (use `slog.DiscardHandler`). Every wait is bounded by a context or timeout.
- Test files sit beside the code as `foo_test.go`, package `foo` for white-box or `foo_test` for black-box. Shared fakes for other packages go in a `footest` package, not exported from `foo`.

## Table-driven tests

```go
func TestParseDuration(t *testing.T) {
	t.Parallel()
	tests := []struct {
		name    string
		in      string
		want    time.Duration
		wantErr error
	}{
		{name: "seconds", in: "30s", want: 30 * time.Second},
		{name: "negative rejected", in: "-1s", wantErr: ErrNegative},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()
			got, err := ParseDuration(tt.in)
			if !errors.Is(err, tt.wantErr) {
				t.Fatalf("ParseDuration(%q) error = %v, want %v", tt.in, err, tt.wantErr)
			}
			if got != tt.want {
				t.Errorf("ParseDuration(%q) = %v, want %v", tt.in, got, tt.want)
			}
		})
	}
}
```

- `t.Run` per case so failures name the case and `-run 'TestX/name'` works.
- Failure messages: function, inputs, got, then want. `t.Error` to continue, `t.Fatal` when continuing is meaningless.
- No `tt := tt` when `go >= 1.22`.
- Keep conditional logic and mock-setup funcs out of table rows. When a table needs branches, split into separate tests.
- `t.Setenv` and `t.Chdir` cannot be used with `t.Parallel()`.

## Helpers and lifecycle

- Helpers that can fail the test call `t.Helper()` first and accept `testing.TB`.
- Setup helpers may `t.Fatal`. Comparison helpers should return an error or a `cmp.Option` rather than take `*testing.T`.
- `t.Cleanup(f)` instead of `defer` inside helpers. Runs LIFO after subtests finish.
- `t.Context()` (1.24+) for anything the test starts. Cancelled just before Cleanup runs.
- `t.TempDir()` for files. `t.ArtifactDir()` (1.26+) for diagnostic output you want to keep with `go test -artifacts`.
- Prefer a real transport against `httptest.NewServer` over a hand-mocked client. Keep doubles small and in a `<pkg>test` package.
- Do not export an interface from the producer package to enable mocking. The consumer defines the interface; the test supplies a fake.

## Assertions: stdlib or testify

The guides disagree. Google bans assertion libraries; most of industry uses testify.

- Follow the repo. If `go.mod` already requires testify, use it consistently: `require` for preconditions that make the rest meaningless, `assert` for independent checks. Never mix styles in one package.
- New repo with no convention: stdlib plus `github.com/google/go-cmp/cmp` for structs and slices. `cmp.Diff(want, got)` with a `-want +got` label. Use `cmpopts.EquateErrors`, `IgnoreUnexported`, `SortSlices` as needed. `cmp` may panic on unsupported types; it is test-only.
- `require.*` and `t.Fatal` must never be called from a goroutine other than the test's.
- Replace `reflect.DeepEqual` with `cmp.Equal` or `slices.Equal`.

## Golden files

- Live under `testdata/`, which the go tool ignores. Compare byte-for-byte.
- Provide `var update = flag.Bool("update", false, "rewrite golden files")` and regenerate with `go test ./... -update`.
- Never golden unstable output: map iteration order, unsorted JSON from maps, timestamps. Canonicalise first.
- Ideal for CLI help text and `--json` output contracts.

## Fuzzing

- `func FuzzX(f *testing.F)` with exactly one `f.Fuzz(func(t *testing.T, in []byte) {...})`. Allowed arg types: string, []byte, bool, ints, uints, floats, rune, byte.
- Seed with `f.Add(...)` and files in `testdata/fuzz/FuzzX/`. Seeds run as regression tests under plain `go test`.
- Run `go test -fuzz=FuzzX -fuzztime=30s`. Commit crashers it writes to `testdata/fuzz/`.
- Targets: parsers, decoders, round-trip invariants, differential checks against a reference. Must be fast, deterministic, free of global state.

## Benchmarks

- `for b.Loop() { ... }` (1.24+). Setup and cleanup are excluded from timing. Do not mix with `b.N`.
- `go test -bench=. -benchmem -run=^$`. Compare with `benchstat`.
- `testing.AllocsPerRun` for allocation assertions. It panics under `t.Parallel()` since 1.25.

## Static analysis

- `go vet ./...` always. Key analyzers: `copylock`, `lostcancel`, `printf`, `loopclosure`, `tests`, `waitgroup` (1.25), `hostport` (1.25), `stdversion`.
- `staticcheck ./...` or golangci-lint v2. v2 config shape: `version: "2"`, `linters.default: standard`, `linters.enable: [...]`, separate `formatters.enable: [gofmt, goimports]`. The v1 `enable-all`/`disable-all` keys are gone.
- Small strict set that good repos converge on: `errcheck`, `govet`, `staticcheck`, `revive`, `errname`, `sloglint`, `testifylint` (if testify), `modernize`, `depguard` banning `github.com/pkg/errors`, `io/ioutil`, `math/rand` for secrets.
- `//nolint:<linter> // reason`. Never a bare `//nolint`.
- `go fix -diff ./...` to preview, `go fix ./...` to apply modernizers (1.26+). Run from a clean tree, then run again to catch cascades.

## Before finishing any Go change

Run, in order, and fix what fails:

```bash
gofmt -l .            # must print nothing
go vet ./...
go fix ./...          # 1.26+; review the diff
go test -race ./...
go mod tidy           # go.mod and go.sum must not change in CI
```

Add `staticcheck ./...` or `golangci-lint run` when the repo has either, and `govulncheck ./...` when dependencies changed. If a step was skipped or failed, report that plainly with the output rather than claiming success.
