# The Context Package and Retry Tracking

## What is Context?

The `context` package solves a specific problem: how do you pass request-scoped values — like a user ID, a trace ID, or a retry counter — through a chain of function calls without adding a new parameter to every function?

The answer is `context.Context`: an immutable value you attach data to and carry through the call chain.

```go
import "context"

ctx := context.Background()                          // empty root context
ctx2 := context.WithValue(ctx, "key", "value")       // new context with attached data
val := ctx2.Value("key").(string)                     // retrieve and type-assert
```

`context.WithValue` returns a *new* context — the original is unchanged. Contexts form a chain: each `WithValue` wraps the previous one.

## Keys Must Not Collide

Using plain strings as context keys is discouraged because two packages could use the same string accidentally. The recommended pattern is to define a private type for keys:

```go
type contextKey int

const (
    Attempts contextKey = iota
    Retry
)
```

`iota` assigns integer values starting from 0: `Attempts = 0`, `Retry = 1`. Because the key type is `contextKey` (a distinct named type), these keys can never collide with keys from other packages — even if they use the same underlying integer value.

## Getting Values from Context

Retrieving a value requires a type assertion because `context.Value` returns `interface{}`:

```go
func GetAttemptsFromContext(r *http.Request) int {
    if attempts, ok := r.Context().Value(Attempts).(int); ok {
        return attempts
    }
    return 0
}

func GetRetryFromContext(r *http.Request) int {
    if retry, ok := r.Context().Value(Retry).(int); ok {
        return retry
    }
    return 0
}
```

The `ok` pattern (comma-ok idiom) is a safe type assertion. If the key doesn't exist in the context, `ok` is `false` and we return the default `0`.

## Using Context in lb()

`lb()` reads the attempt count and returns 503 if the request has already tried too many backends:

```go
func lb(w http.ResponseWriter, r *http.Request) {
    attempts := GetAttemptsFromContext(r)
    if attempts > 3 {
        log.Printf("%s(%s) Max attempts reached\n", r.RemoteAddr, r.URL.Path)
        http.Error(w, "Service not available", http.StatusServiceUnavailable)
        return
    }

    peer := serverPool.GetNextPeer()
    if peer != nil {
        peer.ReverseProxy.ServeHTTP(w, r)
        return
    }
    http.Error(w, "Service not available", http.StatusServiceUnavailable)
}
```

To attach an incremented attempt count to the request context for the next recursive call:

```go
ctx := context.WithValue(r.Context(), Attempts, attempts+1)
lb(w, r.WithContext(ctx))
```

`r.WithContext(ctx)` returns a shallow copy of the request with the new context attached. The original `r` is unchanged.

> **Worked Example**
>
> ```go
> package main
>
> import (
>     "fmt"
>     "net/http"
> )
>
> type contextKey int
>
> const (
>     Attempts contextKey = iota
>     Retry
> )
>
> func GetAttemptsFromContext(r *http.Request) int {
>     if v, ok := r.Context().Value(Attempts).(int); ok {
>         return v
>     }
>     return 0
> }
>
> func handler(w http.ResponseWriter, r *http.Request) {
>     attempts := GetAttemptsFromContext(r)
>     fmt.Fprintf(w, "attempt #%d\n", attempts)
>
>     if attempts < 2 {
>         // simulate a retry by calling handler recursively with incremented context
>         import_ctx := context.WithValue(r.Context(), Attempts, attempts+1)
>         handler(w, r.WithContext(import_ctx))
>     }
> }
> ```
>
> In the real load balancer, the recursive call isn't explicit like this — it happens inside the `ReverseProxy.ErrorHandler` closure you'll build in the next section. The context is the thread that connects the error handler back to `lb()`, passing the retry count forward without changing any function signatures.

## Why Not Just Use a Global Variable?

A global retry counter would be shared across all concurrent requests — one request's retries would interfere with another's. Context values are per-request: each `*http.Request` carries its own `context.Context`, keeping the state isolated.
