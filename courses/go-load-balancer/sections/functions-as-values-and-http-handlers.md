# Functions as Values and HTTP Handlers

## Functions are First-Class Values

In Go, functions are values. You can assign them to variables, pass them as arguments, and return them from other functions — just like integers or strings.

```go
add := func(a, b int) int { return a + b }
fmt.Println(add(2, 3)) // 5
```

You can also define a named function type:

```go
type HandlerFunc func(ResponseWriter, *Request)
```

This is exactly what the standard library does. Any function with the signature `func(http.ResponseWriter, *http.Request)` satisfies `http.HandlerFunc`.

## Building an HTTP Server

The `net/http` package makes building servers straightforward:

```go
import "net/http"

func hello(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello, world!"))
}

func main() {
    http.HandleFunc("/", hello)
    http.ListenAndServe(":3000", nil)
}
```

`http.ListenAndServe` blocks forever, serving requests. The second argument is a `http.Handler`; passing `nil` uses the default mux.

## The Handler Interface

`http.Handler` is a simple interface:

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

`http.HandlerFunc` is a type that *is* a function AND implements this interface:

```go
server := &http.Server{
    Addr:    ":3000",
    Handler: http.HandlerFunc(myFunc),
}
server.ListenAndServe()
```

By wrapping `myFunc` in `http.HandlerFunc(...)`, you convert a plain function into a value that satisfies the `http.Handler` interface. This is the adapter pattern — no struct needed.

## Writing the lb() Stub

Our load balancer's entry point is a function `lb` that the HTTP server calls for every incoming request. For now it returns 503 Service Unavailable (we'll add real routing in the next section):

```go
var serverPool ServerPool

func lb(w http.ResponseWriter, r *http.Request) {
    http.Error(w, "Service not available", http.StatusServiceUnavailable)
}
```

We wire it up in `main()`:

```go
server := http.Server{
    Addr:    fmt.Sprintf(":%d", port),
    Handler: http.HandlerFunc(lb),
}
log.Fatal(server.ListenAndServe())
```

`log.Fatal` prints the error and exits — `ListenAndServe` only returns if something goes wrong (like the port is already in use).

> **Worked Example**
>
> ```go
> package main
>
> import (
>     "fmt"
>     "log"
>     "net/http"
>     "net/url"
> )
>
> type Backend struct {
>     URL   *url.URL
>     Alive bool
> }
>
> type ServerPool struct {
>     backends []*Backend
>     current  uint64
> }
>
> var serverPool ServerPool
>
> func lb(w http.ResponseWriter, r *http.Request) {
>     http.Error(w, "Service not available", http.StatusServiceUnavailable)
> }
>
> func main() {
>     port := 8000
>     server := http.Server{
>         Addr:    fmt.Sprintf(":%d", port),
>         Handler: http.HandlerFunc(lb),
>     }
>     log.Printf("Load Balancer started on port %d\n", port)
>     log.Fatal(server.ListenAndServe())
> }
> ```
>
> Run this, then in another terminal:
> ```
> curl http://localhost:8000
> ```
> You'll get a `503 Service Unavailable` — the server is running, it just has no backends yet. This is the first milestone: we have a live HTTP server. In the next section we'll replace the stub with real traffic forwarding.

## Key Insight

`http.HandlerFunc(lb)` converts the `lb` function into a value that implements `http.Handler`. This is Go's idiomatic way to use functions as objects without needing a class — the function type itself carries the behavior.
