# Closures and Proxy Error Handlers

## What is a Closure?

A **closure** is a function that captures variables from its surrounding scope. The function "closes over" those variables and carries them along even after the outer function returns.

```go
func makeCounter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

c := makeCounter()
fmt.Println(c()) // 1
fmt.Println(c()) // 2
fmt.Println(c()) // 3
```

`count` is defined in `makeCounter`, but the returned function holds a reference to it. Calling `c()` increments the same `count` variable each time.

## Closures Capture Variables, Not Values

Closures capture the *variable itself*, not a snapshot of its value at creation time. This distinction matters in loops:

```go
// Bug: all goroutines print the same final value of i
for i := 0; i < 3; i++ {
    go func() { fmt.Println(i) }()
}

// Fix: pass i as an argument to create a new copy
for i := 0; i < 3; i++ {
    go func(n int) { fmt.Println(n) }(i)
}
```

## The ErrorHandler Field

`httputil.ReverseProxy` has an `ErrorHandler` field:

```go
type ReverseProxy struct {
    // ...
    ErrorHandler func(http.ResponseWriter, *http.Request, error)
}
```

When the backend fails (connection refused, timeout, etc.), the proxy calls `ErrorHandler`. If it's `nil`, it writes a default 502.

We can assign a closure to `ErrorHandler` — one that captures the backend's URL so it can mark the backend as down:

```go
proxy.ErrorHandler = func(w http.ResponseWriter, r *http.Request, e error) {
    log.Printf("[%s] %s\n", serverURL.Host, e.Error())

    retries := GetRetryFromContext(r)
    if retries < 3 {
        select {
        case <-time.After(10 * time.Millisecond):
            ctx := context.WithValue(r.Context(), Retry, retries+1)
            proxy.ServeHTTP(w, r.WithContext(ctx))
        }
        return
    }

    // After 3 retries, mark backend as down
    serverPool.MarkBackendStatus(serverURL, false)

    // Try a different backend
    attempts := GetAttemptsFromContext(r)
    log.Printf("%s(%s) Attempting retry %d\n", r.RemoteAddr, r.URL.Path, attempts)
    ctx := context.WithValue(r.Context(), Attempts, attempts+1)
    lb(w, r.WithContext(ctx))
}
```

The closure captures `serverURL` (the backend's URL) and `proxy` (itself — to retry on the same backend). This is legal: a closure can reference variables in scope, including the proxy it's being assigned to.

## The select Statement

`select` is like a `switch` for channels. It waits until one of its cases is ready:

```go
select {
case <-time.After(10 * time.Millisecond):
    // this case fires after 10ms
    doRetry()
}
```

`time.After` returns a channel that sends a value after the given duration. A `select` with one case is used here purely for the delay — it's Go's way of sleeping for a short time while remaining able to be cancelled (if more cases were added).

> **Worked Example**
>
> Here's the complete `NewBackend` function with the error handler closure wired in:
>
> ```go
> func NewBackend(rawURL string) (*Backend, error) {
>     serverURL, err := url.Parse(rawURL)
>     if err != nil {
>         return nil, err
>     }
>
>     proxy := httputil.NewSingleHostReverseProxy(serverURL)
>
>     proxy.ErrorHandler = func(w http.ResponseWriter, r *http.Request, e error) {
>         log.Printf("[%s] error: %s\n", serverURL.Host, e.Error())
>
>         retries := GetRetryFromContext(r)
>         if retries < 3 {
>             select {
>             case <-time.After(10 * time.Millisecond):
>                 ctx := context.WithValue(r.Context(), Retry, retries+1)
>                 proxy.ServeHTTP(w, r.WithContext(ctx))
>             }
>             return
>         }
>
>         serverPool.MarkBackendStatus(serverURL, false)
>
>         attempts := GetAttemptsFromContext(r)
>         ctx := context.WithValue(r.Context(), Attempts, attempts+1)
>         lb(w, r.WithContext(ctx))
>     }
>
>     return &Backend{URL: serverURL, Alive: true, ReverseProxy: proxy}, nil
> }
> ```
>
> The closure captures `serverURL`, `proxy`, and references the global `serverPool` and `lb`. This is Go's idiomatic way of giving a library type (`ReverseProxy`) custom behavior — assign a closure to its callback field instead of subclassing.

## Chapter 3 Milestone

The load balancer now:
- Protects the `Alive` flag with a `RWMutex`
- Tracks retry and attempt counts per-request via context
- Automatically retries failed backends up to 3 times with a 10ms delay
- Falls over to a different backend after 3 consecutive failures
- Marks dead backends as down so they're skipped by `GetNextPeer`
