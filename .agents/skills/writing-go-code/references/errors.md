# Errors

## Mechanics

- `error` is an interface. Return `error`, never `*MyError`. A nil `*MyError` stored in an `error` is a non-nil interface and `if err != nil` passes. Return the literal `nil`.
- Never discard with `_`. If it is truly safe, write a comment saying why.
- Error strings: lowercase first letter (unless a proper noun), no trailing punctuation. They get embedded in larger messages.
- `errors.Is(err, target)` for sentinel comparison. `==` breaks once anything wraps.
- `errors.AsType[E](err) (E, bool)` on Go 1.26+. Generic, no pre-declared target. Fall back to `errors.As(err, &target)` where `target` is a pointer to the error type (`var e *QueryError; errors.As(err, &e)`). `errors.As` panics on a non-pointer target.
- `errors.Join(errs...)` drops nils and returns nil if all nil. Result implements `Unwrap() []error`. `errors.Unwrap` does NOT unwrap a joined error; `Is` and `As` walk the whole tree.
- `fmt.Errorf` accepts multiple `%w` verbs.
- Never match on error strings. If callers must distinguish, give the error structure.

## Which kind of error

| Caller must match it? | Message | Use |
|---|---|---|
| No | static | `errors.New("...")` |
| No | dynamic | `fmt.Errorf("...", ...)` |
| Yes | static | package-level `var ErrNotFound = errors.New("not found")` |
| Yes | dynamic | custom type `type ValidationError struct{...}` with `Error()` |

- Sentinels are named `ErrX`. Types are named `XError`.
- Expose an identifiable condition by wrapping a sentinel: `fmt.Errorf("card %d: %w", id, ErrNotFound)`. If a function documents that it returns a sentinel, always wrap it, never return it bare, so callers must use `errors.Is` and you can add context later.
- Behaviour interfaces (`interface{ Timeout() bool }`, like `net.Error`) let callers distinguish transient from permanent without knowing concrete types.

## Wrap, hide, or pass through

Wrapping with `%w` makes the inner error part of your API. Three options per call site:

- **Return as-is** when there is nothing useful to add.
- **`%w`** when callers may inspect the cause. Default for most wrapping inside one codebase.
- **`%v`** when the cause is an implementation detail. Use it at storage, RPC and process boundaries. Do not leak `sql.ErrNoRows` or `pgx.ErrNoRows` out of a store package; translate to your own `ErrNotFound`.

Message discipline:

- Context first, `%w` last: `fmt.Errorf("open config: %w", err)`. Printed text then reads outermost to innermost. Exception: a category sentinel may lead, `fmt.Errorf("%w: invalid header", ErrBadRequest)`.
- No "failed to" or "error" filler. Not `"failed to open file: %w"`. Just `"open config: %w"`.
- Do not repeat what the inner error already says. `os.Open` errors include the path.
- Handle once. Return it or log it. Logging at every layer gives duplicate lines; log where the error is finally mapped to a response or exit code.
- Go 1.27 vet flags `fmt.Errorf("...: %w", p)` where `p` is `*E` but only `E` implements `error`. Fix the receiver kind; otherwise `Is`/`As` miss it.

## HTTP handlers

Handlers return errors; one adapter maps them to responses and logs once.

```go
type handlerFunc func(w http.ResponseWriter, r *http.Request) error

func (fn handlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	err := fn(w, r)
	if err == nil {
		return
	}
	status := http.StatusInternalServerError
	switch {
	case errors.Is(err, ErrNotFound):
		status = http.StatusNotFound
	case errors.Is(err, ErrConflict):
		status = http.StatusConflict
	}
	if ve, ok := errors.AsType[*ValidationError](err); ok {
		writeJSON(w, http.StatusBadRequest, ve.Problem())
		return
	}
	if status >= 500 {
		slog.ErrorContext(r.Context(), "request failed", "err", err)
	}
	http.Error(w, http.StatusText(status), status)
}
```

- Map domain errors to status in one place. Errors that leave the server are generic: never send internal error text, and never put secrets, tokens or raw identifiers in the log line (`securing-applications`).
- Always use `r.Context()` downstream. Never `context.Background()` in a handler.
- Go 1.25+ has `http.CrossOriginProtection` for CSRF via Fetch metadata.

## Panics

- Library and service code never panics for ordinary failures.
- Acceptable: `regexp.MustCompile` and `template.Must` at package-var init, and invariant violations that indicate a programmer bug.
- `recover` only works directly inside a deferred function. If a parser uses internal panics, convert to `error` before crossing the package boundary and re-panic on foreign values.
- No blanket recover-to-keep-serving middleware by default. If the repo wants one, log the stack and return 500; do not swallow.
- Type assertions always comma-ok: `v, ok := x.(T)`.
- In tests, `t.Fatal`, not `panic`.

## CLI exit behaviour

Follow the GitHub CLI model: typed sentinel errors drive exit codes and printing. A `SilentError` exits 1 with no message (the command already printed). A `FlagError` prints usage. Scripts and agents rely on stable exit codes and the stdout/stderr split, so never print data to stderr or diagnostics to stdout.
