# Atomic Counter and Round-Robin

## The Problem: Data Races

A **data race** occurs when two goroutines access the same memory location simultaneously and at least one of them is writing. Go's HTTP server runs each request in its own goroutine, so `lb()` is called concurrently. If two goroutines both read and increment a plain `int` counter:

```go
var current int
// goroutine 1 reads current=5, goroutine 2 reads current=5
// both compute 5+1=6 and write back 6
// one increment was lost
```

You can detect races with Go's built-in race detector: `go run -race main.go`.

## Atomic Operations

The `sync/atomic` package provides operations that are **atomic** — they complete as a single indivisible step, invisible to other goroutines:

```go
import "sync/atomic"

var counter uint64
atomic.AddUint64(&counter, 1)           // increment by 1, no race
val := atomic.LoadUint64(&counter)      // read the current value safely
atomic.StoreUint64(&counter, 0)         // write a value safely
```

These functions take a pointer to the variable and operate directly on memory. No lock, no mutex — the CPU provides the atomicity guarantee at the hardware level.

## Implementing NextIndex

`ServerPool.NextIndex()` atomically increments `current` and wraps it to the length of the slice using modulo:

```go
func (s *ServerPool) NextIndex() int {
    return int(atomic.AddUint64(&s.current, uint64(1)) % uint64(len(s.backends)))
}
```

`atomic.AddUint64(&s.current, 1)` increments `s.current` by 1 and returns the *new* value. Modding by `len(s.backends)` gives a valid index between 0 and `len-1`.

## Implementing GetNextPeer

A backend might be down. `GetNextPeer` cycles through the slice starting from `NextIndex` until it finds an alive one:

```go
func (s *ServerPool) GetNextPeer() *Backend {
    next := s.NextIndex()
    l := len(s.backends) + next // traverse up to a full cycle
    for i := next; i < l; i++ {
        idx := i % len(s.backends)
        if s.backends[idx].Alive {
            if i != next {
                atomic.StoreUint64(&s.current, uint64(idx))
            }
            return s.backends[idx]
        }
    }
    return nil
}
```

If `next` points to a dead backend, we check `next+1`, `next+2`, ... wrapping around. When we find an alive one that wasn't the original `next`, we store `idx` as the new current position so the next call continues from there.

> **Worked Example**
>
> Suppose the pool has 3 backends at indices 0, 1, 2. Backend 1 is down.
>
> - Call 1: `NextIndex()` returns 1 (counter was 0, now 1). Backend 1 is dead. Try index 2 — alive. Store 2 as current. Return backend 2.
> - Call 2: `NextIndex()` returns 0 (counter was 2, now 3, 3 % 3 = 0). Backend 0 is alive. Return backend 0.
> - Call 3: `NextIndex()` returns 1. Backend 1 is still dead. Try 2 — alive. Return backend 2.
>
> The updated `lb`:
>
> ```go
> func lb(w http.ResponseWriter, r *http.Request) {
>     peer := serverPool.GetNextPeer()
>     if peer != nil {
>         peer.ReverseProxy.ServeHTTP(w, r)
>         return
>     }
>     http.Error(w, "Service not available", http.StatusServiceUnavailable)
> }
> ```
>
> Now `lb` round-robins across alive backends atomically. Under concurrent load from many clients, every request gets a correct, race-free backend selection.

## Chapter 2 Milestone

The load balancer now:
- Starts an HTTP server
- Forwards requests to backends using `ReverseProxy`
- Selects backends in round-robin order, safely under concurrent load

In the next chapter, we'll add thread safety around the `Alive` flag so health checks can update it while `lb` reads it.
