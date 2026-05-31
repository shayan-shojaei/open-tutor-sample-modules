# Reverse Proxy with httputil

## What is a Reverse Proxy?

A **reverse proxy** sits in front of backend servers. A client sends a request to the proxy; the proxy forwards it to a backend, gets the response, and sends it back to the client. The client never talks to the backend directly.

This is exactly what a load balancer does — it's a reverse proxy with a selection algorithm in front.

## httputil.ReverseProxy

Go's standard library includes a ready-made reverse proxy in `net/http/httputil`:

```go
import "net/http/httputil"

u, _ := url.Parse("http://localhost:8080")
proxy := httputil.NewSingleHostReverseProxy(u)

// proxy implements http.Handler, so you can pass it directly:
http.ListenAndServe(":3000", proxy)
```

`httputil.NewSingleHostReverseProxy(u)` returns an `*httputil.ReverseProxy` that forwards every incoming request to `u`. The response — headers, body, status code — is sent back to the original client transparently.

`ReverseProxy` implements `http.Handler` (it has a `ServeHTTP` method), so you can use it anywhere an `http.Handler` is expected.

## Adding ReverseProxy to Backend

We attach a `ReverseProxy` to each `Backend` so the proxy is created once and reused:

```go
type Backend struct {
    URL          *url.URL
    Alive        bool
    ReverseProxy *httputil.ReverseProxy
}
```

We update `NewBackend` to initialize it:

```go
func NewBackend(rawURL string) (*Backend, error) {
    u, err := url.Parse(rawURL)
    if err != nil {
        return nil, err
    }
    proxy := httputil.NewSingleHostReverseProxy(u)
    return &Backend{URL: u, Alive: true, ReverseProxy: proxy}, nil
}
```

## Forwarding Requests in lb()

Now `lb` can pick a backend and call its proxy:

```go
func lb(w http.ResponseWriter, r *http.Request) {
    // For now, always pick the first backend
    if len(serverPool.backends) == 0 {
        http.Error(w, "Service not available", http.StatusServiceUnavailable)
        return
    }
    peer := serverPool.backends[0]
    peer.ReverseProxy.ServeHTTP(w, r)
}
```

`peer.ReverseProxy.ServeHTTP(w, r)` hands the request to the proxy, which forwards it to the backend and writes the response to `w`.

> **Worked Example**
>
> To test this locally, first start a simple backend server in one terminal:
>
> ```go
> // backend_server.go
> package main
>
> import (
>     "fmt"
>     "log"
>     "net/http"
>     "os"
> )
>
> func main() {
>     port := os.Args[1]
>     http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
>         fmt.Fprintf(w, "Hello from backend on port %s\n", port)
>     })
>     log.Fatal(http.ListenAndServe(":"+port, nil))
> }
> ```
>
> Run: `go run backend_server.go 8081`
>
> Then the load balancer:
>
> ```go
> // main.go (abbreviated)
> func main() {
>     b, _ := NewBackend("http://localhost:8081")
>     serverPool.AddBackend(b)
>
>     server := http.Server{
>         Addr:    ":8000",
>         Handler: http.HandlerFunc(lb),
>     }
>     log.Println("Load Balancer started on :8000")
>     log.Fatal(server.ListenAndServe())
> }
> ```
>
> Now `curl http://localhost:8000` returns `Hello from backend on port 8081`. The load balancer is forwarding real traffic.

## Key Insight

`httputil.NewSingleHostReverseProxy` is a library function that returns a value implementing `http.Handler`. You don't need to write any TCP or HTTP parsing code — the standard library handles all of it. The load balancer's job is solely to *choose* which backend gets the request. The proxy does the forwarding.

In the next section, we'll replace the "always pick index 0" logic with proper atomic round-robin selection.
