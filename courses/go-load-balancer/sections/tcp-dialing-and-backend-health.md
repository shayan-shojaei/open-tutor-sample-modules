# TCP Dialing and Backend Health

## Active vs Passive Health Checks

So far, the load balancer marks a backend as dead only when a request to it fails (active detection). But if all backends are idle and none fail, the load balancer never discovers that a backend went down during a quiet period. When traffic resumes, the first requests to that backend would fail and trigger retries.

**Passive health checks** solve this: probe every backend on a fixed interval to check if it's reachable, regardless of traffic.

## net.DialTimeout

The `net` package provides `net.DialTimeout` for opening a TCP connection with a deadline:

```go
import "net"
import "time"

conn, err := net.DialTimeout("tcp", "localhost:8081", 2*time.Second)
if err != nil {
    // backend is unreachable
    return false
}
defer conn.Close()
return true
```

`"tcp"` is the network type. The second argument is the address in `host:port` form. If the connection succeeds within 2 seconds, the backend is alive. We close the connection immediately — we don't need to send any data. The `defer conn.Close()` ensures the connection is cleaned up even if something panics.

## Why TCP, Not HTTP?

An HTTP health check (sending a real HTTP request) is slower and more complex. All we want to know is: is the backend process running and accepting connections? A successful TCP handshake answers that. If you need to verify application-level health (e.g., the database is connected), you'd use an HTTP `GET /health` endpoint instead.

## Implementing isBackendAlive

```go
func isBackendAlive(u *url.URL) bool {
    timeout := 2 * time.Second
    conn, err := net.DialTimeout("tcp", u.Host, timeout)
    if err != nil {
        log.Println("Site unreachable, error:", err)
        return false
    }
    defer conn.Close()
    return true
}
```

`u.Host` gives us the `host:port` string from the parsed URL — exactly what `DialTimeout` expects.

## The HealthCheck Method

`ServerPool.HealthCheck` iterates all backends, probes each, and updates their `Alive` flag:

```go
func (s *ServerPool) HealthCheck() {
    for _, b := range s.backends {
        alive := isBackendAlive(b.URL)
        b.SetAlive(alive)
        status := "up"
        if !alive {
            status = "down"
        }
        log.Printf("%s [%s]\n", b.URL, status)
    }
}
```

Because `SetAlive` uses a write lock, health checks and incoming requests can run concurrently without data races.

> **Worked Example**
>
> ```go
> package main
>
> import (
>     "fmt"
>     "log"
>     "net"
>     "net/url"
>     "time"
> )
>
> func isBackendAlive(u *url.URL) bool {
>     conn, err := net.DialTimeout("tcp", u.Host, 2*time.Second)
>     if err != nil {
>         return false
>     }
>     defer conn.Close()
>     return true
> }
>
> func main() {
>     // Try a likely-unreachable address
>     u1, _ := url.Parse("http://localhost:19999")
>     fmt.Printf("localhost:19999 alive: %v\n", isBackendAlive(u1))
>
>     // Try your own machine's HTTP port (if something is running there)
>     u2, _ := url.Parse("http://localhost:8080")
>     fmt.Printf("localhost:8080 alive: %v\n", isBackendAlive(u2))
> }
> ```
>
> Port 19999 is likely not running anything, so `isBackendAlive` returns `false` quickly (connection refused). Port 8080 depends on your machine — try starting a backend server there to see `true`.
>
> The key detail: `defer conn.Close()` runs when `isBackendAlive` returns. This is essential — if you forget to close the connection, every health check cycle leaks a file descriptor. Eventually the OS runs out and the process dies.

## error as a Value

Notice how `net.DialTimeout` returns `(net.Conn, error)` — Go's idiomatic error handling. Errors are plain values, not exceptions. You check them with `if err != nil` and handle them immediately. This makes the control flow explicit and keeps error handling close to where the failure occurs.
