# Slices and the ServerPool

## Slices in Go

A **slice** is a dynamically-sized view into an array. It's Go's workhorse collection type. You can think of it as three fields under the hood: a pointer to an underlying array, a length, and a capacity.

```go
backends := []string{"localhost:8080", "localhost:8081", "localhost:8082"}
fmt.Println(len(backends)) // 3
fmt.Println(backends[0])   // localhost:8080
```

You rarely work with raw arrays in Go. Slices are used almost everywhere instead.

## Appending to a Slice

The built-in `append` function adds elements to a slice:

```go
var servers []string
servers = append(servers, "localhost:8080")
servers = append(servers, "localhost:8081")
fmt.Println(servers) // [localhost:8080 localhost:8081]
```

Always reassign the result of `append` — it may return a new underlying array if the capacity was exceeded.

## Iterating with range

`range` gives you the index and value for each element:

```go
for i, s := range servers {
    fmt.Printf("[%d] %s\n", i, s)
}
```

Use `_` to discard the index when you don't need it:

```go
for _, s := range servers {
    fmt.Println(s)
}
```

## Slices of Pointers

In our load balancer, we store `*Backend` pointers — not `Backend` values — so that multiple parts of the program can reference and mutate the same backend without copying:

```go
var backends []*Backend
b, _ := NewBackend("http://localhost:8080")
backends = append(backends, b)
```

All pointers in the slice point to the same heap-allocated structs. If one goroutine marks a backend as dead (`b.Alive = false`), every other part of the program sees the change immediately.

## The ServerPool Struct

`ServerPool` is the central coordinator of the load balancer. It holds the slice of backends and a counter used to track which backend to send traffic to next:

```go
type ServerPool struct {
    backends []*Backend
    current  uint64
}

func (s *ServerPool) AddBackend(b *Backend) {
    s.backends = append(s.backends, b)
}
```

`current uint64` will be incremented atomically (in a later section) to implement round-robin. For now, it's just a field that holds our place.

> **Worked Example**
>
> ```go
> package main
>
> import (
>     "fmt"
>     "log"
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
> func (s *ServerPool) AddBackend(b *Backend) {
>     s.backends = append(s.backends, b)
> }
>
> func NewBackend(rawURL string) (*Backend, error) {
>     u, err := url.Parse(rawURL)
>     if err != nil {
>         return nil, err
>     }
>     return &Backend{URL: u, Alive: true}, nil
> }
>
> func main() {
>     pool := &ServerPool{}
>
>     urls := []string{
>         "http://localhost:8081",
>         "http://localhost:8082",
>         "http://localhost:8083",
>     }
>     for _, rawURL := range urls {
>         b, err := NewBackend(rawURL)
>         if err != nil {
>             log.Fatal(err)
>         }
>         pool.AddBackend(b)
>     }
>
>     fmt.Printf("Pool has %d backends:\n", len(pool.backends))
>     for i, b := range pool.backends {
>         fmt.Printf("  [%d] %s alive=%v\n", i, b.URL.Host, b.Alive)
>     }
> }
> ```
>
> Output:
> ```
> Pool has 3 backends:
>   [0] localhost:8081 alive=true
>   [1] localhost:8082 alive=true
>   [2] localhost:8083 alive=true
> ```
>
> The `ServerPool` is a pointer (`&ServerPool{}`) so that `AddBackend` can mutate its `backends` slice. The slice grows on each `append` call, and since `backends` is a field on the struct (not a local variable), the pointer receiver ensures the growth is visible outside the method.

## Milestone Reached

At this point, the data model for the load balancer is complete. You can instantiate backends, parse their URLs, group them into a pool, and iterate over them. In the next chapter, we'll wire these structs into a real HTTP server and start forwarding requests.
