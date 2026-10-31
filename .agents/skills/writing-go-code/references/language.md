# Language nuances

## Version gates

The `go` line in `go.mod` is a minimum and sets language semantics. `go mod init` on a 1.26+ toolchain writes one minor version BELOW the toolchain (a 1.27 toolchain writes `go 1.26.0`). Raise it deliberately with `go get go@1.27.0` when a newer feature is needed. Never use a feature newer than the `go` line.

| Needs `go >=` | Feature |
|---|---|
| 1.21 | `min`, `max`, `clear` builtins; `slices`, `maps`, `cmp` packages; `log/slog` |
| 1.22 | Per-iteration loop variables; `for i := range n`; `net/http` method and wildcard routing patterns |
| 1.23 | Range-over-func iterators, `iter.Seq`; unbuffered timer channels |
| 1.24 | `tool` directive in go.mod; `testing.B.Loop`; `t.Context()`; `t.Chdir`; `os.Root`; `omitzero` JSON tag; `strings.Lines`, `SplitSeq`; `weak`; `runtime.AddCleanup`; generic type aliases |
| 1.25 | `sync.WaitGroup.Go`; `testing/synctest` GA; `t.Attr`; container-aware GOMAXPROCS; `http.CrossOriginProtection`; vet `waitgroup` and `hostport` |
| 1.26 | `new(expr)`; `errors.AsType`; `go fix` modernizers; Green Tea GC default; `slog.NewMultiHandler`; `t.ArtifactDir`; `signal.NotifyContext` cause; ServeMux trailing-slash redirect is 307 |
| 1.27 | Generic methods (own type params); `encoding/json/v2` and `jsontext`; `synctest.Sleep`; `httptest.NewTestServer`; goroutineleak profile GA; `go mod tidy` merges require blocks; vet `%w` pointer check |

The Go FAQ still says methods cannot have type parameters. It is stale as of 1.27. Generic methods still cannot implement interface methods, and interface methods cannot be generic.

## Loop variables

- `go >= 1.22`: each iteration has its own `i` and `v`. Closures and goroutines capture the iteration's value. Delete `v := v` and `tt := tt`. `go fix` does this.
- `go < 1.22`: the old shared variable. Keep the copy.
- `for i := range n` runs zero times when `n <= 0`.
- Parallel subtests that passed by accident under old semantics may start failing after a bump. That is a real bug surfacing.

## nil interfaces vs nil pointers

An interface is nil only when both its type and value are unset. A `(*T)(nil)` in an interface is non-nil. This bites `error` returns and `any` parameters. Pointer-to-interface (`*io.Reader`) is almost never what you want.

Comparing two interface values whose dynamic types are non-comparable (two slices, two maps) panics at runtime. Using such a value as a map key panics.

## Slices

- A slice is (pointer, len, cap). Reslicing and passing share the backing array.
- Always assign `append`: `s = append(s, x)`. Two appends from the same base slice with spare capacity overwrite each other.
- Returning a small sub-slice of a large buffer retains the whole buffer. `slices.Clone` it.
- Nil slice works with `len`, `cap`, `range`, `append`. Prefer it over `[]T{}` except where JSON must emit `[]` not `null`.
- Copy at boundaries: when storing a caller's slice in a struct, and when returning an internal slice.
- Arrays are values and copy on assignment. Use slices.
- Prefer `slices.Contains`, `slices.Index`, `slices.Sort`, `slices.SortFunc`, `slices.Equal`, `slices.Reverse`, `slices.Compact` over hand loops.

## Maps

- Missing key yields the zero value. Use `v, ok := m[k]` when absence matters.
- Concurrent read and write is a fatal runtime error, not a race report. Guard with `sync.RWMutex` or single ownership.
- `m[k].field = v` does not compile; map values are not addressable. Store `*T`, or read, modify, write back.
- Iteration order is random. `slices.Sorted(maps.Keys(m))` when order matters.
- `clear(m)` empties it. `clear(s)` on a slice zeroes elements without changing `len`.

## Shadowing

- `:=` inside `if`, `for`, `switch`, or a nested block declares a NEW variable. The outer one is unchanged afterwards. Classic bug:

```go
ctx := context.Background()
if timeout > 0 {
	ctx, cancel := context.WithTimeout(ctx, timeout) // shadows; outer ctx untouched
	defer cancel()
}
```

Declare `var cancel context.CancelFunc` outside and assign with `=`.

- `:=` in the same scope reuses existing variables only when at least one new variable is on the left.
- In goroutines, `_, err = f()` writing a shared outer `err` is a data race. Use `:=`.
- Do not shadow predeclared names: `err`, `len`, `new`, `string`, `error`. Do not name variables after imported packages (`url`, `path`).

## Defer

- Arguments are evaluated at the `defer` statement; the call runs at function return, LIFO.
- `defer` in a loop runs at function exit, not per iteration. Wrap the body in a func.
- Defer right after the successful acquire: `f, err := os.Open(p); if err != nil {...}; defer f.Close()`.
- A deferred closure may read and set named results. That is the one legitimate use of named results. Do not name results just to enable naked returns.

## Struct and embedding

- Embedding is composition with method promotion, not inheritance. Promoted methods run with the inner value as receiver.
- Always use field names in composite literals of structs from other packages.
- Tag every field of a struct that is marshalled. Unexported fields are ignored by `encoding/json`.
- `omitzero` (1.24+) omits a zero `time.Time`. `omitempty` does not.
- Enums with `iota`; start at `iota + 1` when zero should mean unset.
- Option struct when most callers set options; functional options when most need none. Never put a context in an options struct.

## Generics

- Use for code repeated per type: containers, slice and map helpers, shared algorithms. Prefer `slices`, `maps`, `cmp` before writing your own.
- Do not use when an interface does the job (`io.Reader`, not `Read[T io.Reader]`), for speed, or when behaviour differs per type.
- Write the concrete version first; add type parameters when duplication appears.
- Pass a comparison func rather than requiring a `Compare` method.
- No generic DSLs, no generic error-handling frameworks.

## Iterators (1.23+)

- Shapes: `iter.Seq[V]` is `func(yield func(V) bool)`, `iter.Seq2[K, V]` adds a key.
- Contract: when `yield` returns false, stop promptly and never call `yield` again.
- Expose as a method returning the iterator: `func (s *Set[E]) All() iter.Seq[E]`, consumed with `for v := range s.All()`.
- `iter.Pull(seq)` returns `(next, stop)`; always `defer stop()`. Use only when a push loop cannot express it.
- Do not wrap a plain slice loop in an iterator. A loop is usually clearer.

## Strings, bytes, runes

- Strings are read-only byte sequences, not guaranteed UTF-8. `len(s)` is bytes, `s[i]` is a byte, `for i, r := range s` decodes runes with byte offsets.
- `rune` is `int32`. A user-visible character may be several runes. Go does not normalise.
- Truncate user text by rune or with `utf8`, never by byte index.
- `%q` to print strings safely, `%x` to inspect bad UTF-8.
- `[]byte(s)` and `string(b)` copy. Use `strings.Builder` or `bytes.Buffer` for assembly.
- `strings.Cut` instead of `Index` plus slicing.
- Inside `String()`, never `Sprintf("%s", receiver)`. Infinite recursion.
- Integer overflow wraps silently. Check bounds before narrowing conversions on untrusted input.
- `strconv` for primitive conversions, not `fmt`.

## Time

- `time.Time` for instants, `time.Duration` for spans. Never bare ints. On the wire, put the unit in the name (`TimeoutMillis`) or use RFC 3339.
- Inject a clock (`now func() time.Time`) or use `synctest`. Never read the wall clock or `time.Sleep` in tests.
- Reuse `time.NewTimer` with `Reset`/`Stop` instead of `time.After` in loops.
- `math/rand.Seed` is a no-op since 1.24.

## slog

- `slog.Info("msg", "key", val)` alternates key and value. Vet catches odd counts.
- `logger.With("request_id", id)` for repeated attributes.
- `InfoContext`, `ErrorContext` so a Handler can pull trace IDs from ctx.
- `LogAttrs` with `slog.String(...)` on hot paths.
- Log lines never carry secrets, tokens or raw identifiers (`securing-applications`). Implement `LogValuer` to redact.
- `slog.DiscardHandler` (1.24+) in tests. `slog.NewMultiHandler` (1.26+) to fan out.
- Test custom handlers with `testing/slogtest`.

## JSON

- Since 1.27 `encoding/json` runs on `encoding/json/v2` internals. Error text changed, invalid UTF-8 and duplicate names are stricter in v2 APIs. Never match on JSON error strings.
- Use `encoding/json/v2` directly for new code only when the module's `go` line is 1.27+ and the repo has adopted it.
