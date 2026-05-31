# Goroutines and Tickers

## Goroutines

A **goroutine** is a lightweight, concurrently-executing function. You start one with the `go` keyword:

```go
go doSomething()
go func() {
    fmt.Println("running concurrently")
}()
```

Goroutines are not OS threads — the Go runtime multiplexes many goroutines onto a small pool of OS threads. Starting one costs only a few kilobytes of stack, so you can run thousands simultaneously.

The key characteristic: goroutines run *concurrently* with the code that spawned them. The spawning code does not wait for the goroutine to finish.

```go
func main() {
    go fmt.Println("from goroutine") // may or may not print before main exits
    fmt.Println("from main")
    // if main returns here, the goroutine is killed
}
```

## time.Ticker

`time.NewTicker(d)` returns a `*time.Ticker` that sends the current time on its `C` channel every `d` duration:

```go
t := time.NewTicker(20 * time.Second)
for {
    tick := <-t.C  // blocks until the next tick
    fmt.Println("tick at", tick)
}
```

`<-t.C` is a receive from a channel. It blocks the current goroutine until the ticker fires. This is how you implement periodic background work in Go — no `sleep` loops needed.

## select with Channels

`select` waits for the first of several channel operations to be ready:

```go
select {
case <-ticker.C:
    // fires every 20s
case <-quit:
    // fires when quit channel is closed
    return
}
```

In our health checker we only have one case, so `select` behaves like a simple channel receive. The benefit of `select` over a direct `<-t.C` is extensibility: you can add a `quit` channel later to cleanly shut down the goroutine.

## Implementing the Health Check Loop

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

func healthCheck() {
    t := time.NewTicker(20 * time.Second)
    for {
        select {
        case <-t.C:
            log.Println("Starting health check...")
            serverPool.HealthCheck()
            log.Println("Health check completed")
        }
    }
}
```

In `main()`, we start this as a background goroutine:

```go
go healthCheck()
```

`healthCheck` runs forever in the background, waking every 20 seconds to probe all backends. It doesn't block `main()` or the HTTP server — they all run concurrently.

> **Worked Example**
>
> ```go
> package main
>
> import (
>     "fmt"
>     "time"
> )
>
> func periodicLogger(interval time.Duration, label string) {
>     t := time.NewTicker(interval)
>     for {
>         select {
>         case tick := <-t.C:
>             fmt.Printf("[%s] tick at %s\n", label, tick.Format("15:04:05"))
>         }
>     }
> }
>
> func main() {
>     go periodicLogger(1*time.Second, "fast")
>     go periodicLogger(3*time.Second, "slow")
>
>     // Let the goroutines run for 5 seconds
>     time.Sleep(5 * time.Second)
>     fmt.Println("done")
> }
> ```
>
> Both goroutines run independently. "fast" fires every second, "slow" fires every 3 seconds. Neither blocks the other, and `main` sleeps while they run. When `main` returns, all goroutines are killed.
>
> In the real load balancer, `main` doesn't sleep — it blocks on `server.ListenAndServe()`, which keeps the program alive while `healthCheck` runs in the background.

## Goroutine Lifetime

A goroutine lives as long as its function is running. Background goroutines with `for { select { ... } }` loops run until the process exits (or until a quit channel tells them to stop). The health checker is meant to run for the lifetime of the load balancer, so the infinite loop is intentional.
