# Flag Parsing and Wiring main()

## The flag Package

The `flag` package parses command-line arguments. It's the standard way to give Go programs configurable options at startup:

```go
import "flag"

port := flag.Int("port", 3000, "Port to serve on")
backends := flag.String("backends", "", "Comma-separated list of backend URLs")
flag.Parse()

fmt.Println(*port)    // 3000 (or whatever was passed with --port)
fmt.Println(*backends) // "" (or what was passed with --backends)
```

`flag.Int` and `flag.String` return *pointers* — the value is filled in after `flag.Parse()` is called. Notice the `*` when reading the values.

You run the program like:
```
go run main.go --port 8000 --backends http://localhost:8081,http://localhost:8082
```

## Parsing the Backend List

The `--backends` flag is a comma-separated string. `strings.Split` breaks it into individual URLs:

```go
import "strings"

backendList := strings.Split(*backends, ",")
for _, rawURL := range backendList {
    url := strings.TrimSpace(rawURL)
    if url == "" {
        continue
    }
    // create and add backend...
}
```

`strings.TrimSpace` removes accidental whitespace around URLs.

## Putting It All Together

Here is the complete `main()` function that assembles every piece from the previous sections:

```go
func main() {
    var serverList string
    var port int
    flag.StringVar(&serverList, "backends", "", "Comma-separated backends: http://host1,http://host2")
    flag.IntVar(&port, "port", 3000, "Port to serve on")
    flag.Parse()

    if serverList == "" {
        log.Fatal("Please provide one or more backends via --backends")
    }

    for _, tok := range strings.Split(serverList, ",") {
        serverURL, err := url.Parse(strings.TrimSpace(tok))
        if err != nil {
            log.Fatal(err)
        }
        proxy := httputil.NewSingleHostReverseProxy(serverURL)
        // attach error handler closure (from closures section)
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
            serverPool.MarkBackendStatus(serverURL, false)
            attempts := GetAttemptsFromContext(r)
            ctx := context.WithValue(r.Context(), Attempts, attempts+1)
            lb(w, r.WithContext(ctx))
        }
        serverPool.AddBackend(&Backend{
            URL:          serverURL,
            Alive:        true,
            ReverseProxy: proxy,
        })
        log.Printf("Configured backend: %s\n", serverURL)
    }

    server := http.Server{
        Addr:    fmt.Sprintf(":%d", port),
        Handler: http.HandlerFunc(lb),
    }

    go healthCheck()

    log.Printf("Load Balancer started at :%d\n", port)
    if err := server.ListenAndServe(); err != nil {
        log.Fatal(err)
    }
}
```

`main()` is deliberately thin: it parses input, builds the dependency graph, and hands off. All actual logic lives in the functions and methods built in previous sections.

> **Worked Example — Running the Finished Load Balancer**
>
> Start two simple backend servers (in separate terminals):
> ```bash
> # terminal 1
> go run backend_server.go 8081
>
> # terminal 2
> go run backend_server.go 8082
> ```
>
> Start the load balancer:
> ```bash
> go run main.go --port 8000 --backends http://localhost:8081,http://localhost:8082
> ```
>
> Send requests:
> ```bash
> curl http://localhost:8000   # → Hello from backend on port 8081
> curl http://localhost:8000   # → Hello from backend on port 8082
> curl http://localhost:8000   # → Hello from backend on port 8081
> ```
>
> Kill backend 8081 (Ctrl+C in terminal 1). Within 20 seconds the health check marks it down. After that:
> ```bash
> curl http://localhost:8000   # → Hello from backend on port 8082
> curl http://localhost:8000   # → Hello from backend on port 8082
> ```
>
> Restart 8081. It's marked alive on the next health check cycle and traffic returns to it.
>
> You've built a complete, production-grade round-robin load balancer with health checking, retry logic, and zero external dependencies — using only Go's standard library.

## Final Chapter Milestone

The finished load balancer:
- Parses backends from a CLI flag
- Round-robins requests across alive backends atomically
- Retries failed requests up to 3 times with a delay
- Falls over to a different backend after exhausted retries
- Marks dead backends down in real time
- Passively probes all backends every 20 seconds via TCP health checks
- Is race-condition-free under concurrent load
