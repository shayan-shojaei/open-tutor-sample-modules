# Pointers and URL Parsing

## What is a Pointer?

A **pointer** holds the memory address of another variable. Instead of storing a value, it stores *where* the value lives.

```go
x := 42
p := &x   // p is a *int — it holds the address of x
fmt.Println(p)   // e.g., 0xc000014088 (some memory address)
fmt.Println(*p)  // 42 — dereference: follow the pointer to get the value
*p = 100
fmt.Println(x)   // 100 — we changed x through the pointer
```

The `&` operator takes the address of a variable. The `*` operator dereferences a pointer (follows it to the value).

## Why Use Pointers?

Two main reasons:

1. **Mutation**: passing a pointer lets a function modify the caller's data (as we saw with pointer receivers).
2. **Sharing without copying**: large structs are expensive to copy. Passing `*Backend` passes 8 bytes (a memory address) instead of the whole struct.

A pointer's zero value is `nil`. Dereferencing a nil pointer panics at runtime, so nil checks matter:

```go
var p *int
fmt.Println(p)  // <nil>
fmt.Println(*p) // panic: runtime error: invalid memory address
```

## The url.URL Type

The standard library's `net/url` package provides a `url.URL` struct that parses and represents a URL:

```go
import "net/url"

u, err := url.Parse("http://localhost:8080")
if err != nil {
    log.Fatal(err)
}
fmt.Println(u.Host)   // localhost:8080
fmt.Println(u.Scheme) // http
fmt.Println(u.Path)   // (empty for this URL)
```

`url.Parse` returns `*url.URL` (a pointer) because the struct is moderately large and the standard library always hands you a pointer to heap-allocated data. This is the canonical Go pattern for constructor-like functions: return a pointer to the newly created value.

## Building NewBackend

We can now write a proper constructor for our load balancer's `Backend`:

```go
func NewBackend(rawURL string) (*Backend, error) {
    u, err := url.Parse(rawURL)
    if err != nil {
        return nil, err
    }
    return &Backend{URL: u, Alive: true}, nil
}
```

Note `return nil, err` — returning `nil` for the pointer when there's an error is idiomatic Go. The caller checks `err` before using the returned value.

The `Backend` struct's `URL` field is now `*url.URL` instead of `string`:

```go
type Backend struct {
    URL   *url.URL
    Alive bool
}
```

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
> func NewBackend(rawURL string) (*Backend, error) {
>     u, err := url.Parse(rawURL)
>     if err != nil {
>         return nil, err
>     }
>     return &Backend{URL: u, Alive: true}, nil
> }
>
> func (b Backend) String() string {
>     return fmt.Sprintf("%s [alive=%v]", b.URL.Host, b.Alive)
> }
>
> func main() {
>     b, err := NewBackend("http://localhost:8080")
>     if err != nil {
>         log.Fatal(err)
>     }
>     fmt.Println(b)            // localhost:8080 [alive=true]
>     fmt.Println(b.URL.Scheme) // http
>     fmt.Println(b.URL.Host)   // localhost:8080
> }
> ```
>
> `NewBackend` returns `*Backend` — a pointer to a heap-allocated struct. The caller holds a pointer and doesn't need to copy the whole struct to pass it around. This is exactly how the load balancer will work: `[]*Backend` — a slice of pointers, not a slice of copies.

## nil Checks in Practice

When a function can fail, always check the error before using the returned pointer:

```go
b, err := NewBackend("not a valid url ://")
if err != nil {
    // b is nil here — don't use it
    log.Fatal(err)
}
// safe to use b here
fmt.Println(b.URL.Host)
```

In the next section we'll collect multiple `*Backend` values into a slice to build the `ServerPool`.
