# Concurrency

## Choosing a primitive

- Prefer designs where every goroutine's lifetime is obvious from the code. "Share memory by communicating" is guidance, not a ban on mutexes: a `sync.Mutex` is correct for a counter or cache.
- `sync.Mutex` as a named field, never embedded (embedding leaks `Lock`/`Unlock` into the API). Zero value is unlocked. Unlock with `defer`. Never copy after first use; vet `copylock` catches it.
- Typed atomics from stdlib: `atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`. Not `go.uber.org/atomic`, which predates them.
- `sync.Map` is for append-only caches or disjoint key sets. Not a general concurrent map. Use `sync.RWMutex` around a map, or give the map one owning goroutine.
- `sync.OnceFunc`, `OnceValue`, `OnceValues` for lazy init. If `f` panics, every call re-panics.
- `TryLock` is usually a design smell.
- Channel buffers: unbuffered or size 1 unless you can name the reason. Size to a known worker count when building a semaphore.

## WaitGroup

Go 1.25+ (check `go.mod`):

```go
var wg sync.WaitGroup
for _, item := range items {
	wg.Go(func() { process(ctx, item) })
}
wg.Wait()
```

Older modules: `wg.Add(1)` BEFORE the `go` statement, `defer wg.Done()` as the first line inside. `Add` inside the goroutine races with `Wait`. Vet `waitgroup` (1.25+) flags this.

`wg.Go` does not collect errors or cancel siblings. For that use errgroup.

## errgroup

`golang.org/x/sync/errgroup`.

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8)
results := make([]Result, len(inputs))
for i, in := range inputs {
	g.Go(func() error {
		r, err := fetch(ctx, in)
		if err != nil {
			return fmt.Errorf("fetch %s: %w", in.ID, err)
		}
		results[i] = r
		return nil
	})
}
if err := g.Wait(); err != nil {
	return nil, err
}
```

- `Wait` returns the first non-nil error. The derived `ctx` is cancelled when the first function fails or when `Wait` returns, whichever is first. Do not reuse it after `Wait`.
- Functions must observe `ctx.Done()` or cancellation does nothing.
- `SetLimit` before launching, never while goroutines run. `TryGo` is the non-blocking form.
- Write results into a pre-sized slice by index rather than appending under a mutex.
- Do not reuse a Group for a different batch of work.

## Pipelines and fan-out

- Bounded parallelism: a fixed pool of N workers reading one channel, or errgroup with `SetLimit`. Never one goroutine per item on unbounded input.
- Only the sender closes a channel. The stage that owns `out` does `defer close(out)`. With several producers, a coordinator does `wg.Wait(); close(out)`.
- Every send in a cancellable goroutine:

```go
select {
case out <- v:
case <-ctx.Done():
	return ctx.Err()
}
```

- A goroutine producing a single result that the caller may abandon must send on a buffered channel of size 1, or it leaks forever.
- Sending on a closed channel panics. Closing twice panics. Receivers never close.
- A nil channel blocks forever in `select`. Set a channel variable to nil to disable that case.
- `select` with `default` is non-blocking. Do not spin on it.
- CPU-bound fan-out width: `runtime.GOMAXPROCS(0)`. Since 1.25 it is container-aware; do not import `automaxprocs`.
- Read the error channel only after draining results, or use a size-1 buffered error channel so the sender never blocks.

## Context

- First parameter, named `ctx`. Never in a struct field. No custom Context types.
- `context.Background()` only in `main`, `init`, the top of an incoming request, and tests (prefer `t.Context()` on 1.24+).
- `defer cancel()` immediately after `WithCancel`, `WithTimeout`, `WithDeadline`. Vet `lostcancel` checks the obvious cases.
- Cancellation is advisory. Loops and blocking calls must check `ctx.Done()` and return `ctx.Err()`.
- Values: only request-scoped data that crosses API boundaries (request ID, principal, tenant). Never optional parameters. Unexported key type, typed `FromContext`/`WithX` accessors in one small package.
- `signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)` for shutdown; `context.Cause(ctx)` names the signal on 1.26+.
- Do not document "returns when ctx is cancelled". That is assumed.

## Goroutine leaks

- A goroutine blocked forever is never collected and pins its stack and heap.
- Every `go` statement needs: an exit condition (ctx, closed channel, done signal) and something that waits for it. No fire-and-forget.
- Go 1.27 ships `/debug/pprof/goroutineleak` for production diagnosis.
- In tests, `go.uber.org/goleak` or `testing/synctest` (a deadlocked bubble fails the test).

## Race detector

- `go test -race ./...` in CI always. Only executed paths are checked, so also run a `-race` binary under realistic load when it matters.
- Common finds: shared `err` assigned with `=` inside goroutines (declare with `:=`), unprotected maps, unsynchronised counters, a send racing a close, captured loop variables in pre-1.22 modules.
- Exit code 66 on a detected race.

## Testing concurrent code

- `testing/synctest` is GA since 1.25: `synctest.Test(t, func(t *testing.T) {...})` runs in a bubble with a fake clock that advances only when every goroutine is durably blocked. `synctest.Wait()` waits for that state. `synctest.Sleep(d)` (1.27) is sleep plus wait. No real `time.Sleep` in tests.
- Not durably blocking inside a bubble: mutex waits, real network I/O, syscalls, channels created outside the bubble. Use `net.Pipe` or fakes. `httptest.NewTestServer` (1.27) gives an in-memory network.
- Never call `t.Fatal`, `t.FailNow` or `require.*` from a goroutine other than the test's own. Use `t.Error` and return.
