# Mutexes and the Alive Flag

## The Problem with the Alive Field

The `Alive bool` field on `Backend` is read by `GetNextPeer` (potentially thousands of times per second) and written by health checks (occasionally). If a health check goroutine writes `b.Alive = false` at the same moment `GetNextPeer` reads `b.Alive`, that's a data race — even for a single boolean.

Atomic operations work for primitive integers, but not for arbitrary struct fields. For protecting general fields, Go provides **mutexes**.

## sync.Mutex

A `sync.Mutex` allows only one goroutine to hold the lock at a time:

```go
import "sync"

var mu sync.Mutex

mu.Lock()
// only one goroutine here at a time
sharedData = newValue
mu.Unlock()
```

Always call `Unlock` after `Lock`. A deferred unlock is the idiomatic pattern:

```go
mu.Lock()
defer mu.Unlock()
// do work — Unlock called automatically when function returns
```

## sync.RWMutex

When reads are far more frequent than writes, `sync.RWMutex` is more efficient:

- `RLock()` / `RUnlock()` — many goroutines can hold a read lock simultaneously
- `Lock()` / `Unlock()` — only one goroutine can hold the write lock, and no readers can run concurrently

```go
var mu sync.RWMutex

// Writer (exclusive access)
mu.Lock()
alive = false
mu.Unlock()

// Reader (concurrent access with other readers)
mu.RLock()
val := alive
mu.RUnlock()
```

In our load balancer: `IsAlive()` is called constantly by every incoming request, but `SetAlive()` is called only by health checks. `RWMutex` is the right fit.

## Updating Backend

We embed the mutex directly in the `Backend` struct (no pointer needed — embedded value types work correctly):

```go
type Backend struct {
    URL          *url.URL
    Alive        bool
    mux          sync.RWMutex
    ReverseProxy *httputil.ReverseProxy
}

func (b *Backend) SetAlive(alive bool) {
    b.mux.Lock()
    b.Alive = alive
    b.mux.Unlock()
}

func (b *Backend) IsAlive() bool {
    b.mux.RLock()
    alive := b.Alive
    b.mux.RUnlock()
    return alive
}
```

`GetNextPeer` should now call `b.IsAlive()` instead of accessing `b.Alive` directly:

```go
if s.backends[idx].IsAlive() {
    // ...
}
```

> **Worked Example**
>
> ```go
> package main
>
> import (
>     "fmt"
>     "sync"
>     "net/url"
> )
>
> type Backend struct {
>     URL  *url.URL
>     Alive bool
>     mux  sync.RWMutex
> }
>
> func (b *Backend) SetAlive(alive bool) {
>     b.mux.Lock()
>     b.Alive = alive
>     b.mux.Unlock()
> }
>
> func (b *Backend) IsAlive() bool {
>     b.mux.RLock()
>     defer b.mux.RUnlock()
>     return b.Alive
> }
>
> func main() {
>     u, _ := url.Parse("http://localhost:8081")
>     b := &Backend{URL: u, Alive: true}
>
>     var wg sync.WaitGroup
>
>     // Simulate concurrent reads
>     for i := 0; i < 5; i++ {
>         wg.Add(1)
>         go func() {
>             defer wg.Done()
>             fmt.Println("alive:", b.IsAlive())
>         }()
>     }
>
>     // Simulate a health check write
>     wg.Add(1)
>     go func() {
>         defer wg.Done()
>         b.SetAlive(false)
>     }()
>
>     wg.Wait()
>     fmt.Println("final:", b.IsAlive()) // false
> }
> ```
>
> Run this with `go run -race main.go` — no race reported. Without the mutex, the race detector would fire. `sync.WaitGroup` (used here to wait for all goroutines) is itself a useful synchronization primitive worth knowing, though the load balancer uses it only in tests.

## Rule of Thumb

- Use `sync.Mutex` when reads and writes are roughly equal, or the critical section is complex.
- Use `sync.RWMutex` when reads dominate and writes are rare.
- Use `sync/atomic` when you're only incrementing or loading/storing a primitive integer.
