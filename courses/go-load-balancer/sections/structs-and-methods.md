# Structs and Methods

## Grouping Data with Structs

In Go, a **struct** is a named collection of fields. It lets you bundle related data together under one type. You define a struct with the `type` keyword:

```go
type Backend struct {
    URL   string
    Alive bool
}
```

This defines a new type `Backend` with two fields. You create values of this type like this:

```go
b := Backend{URL: "http://localhost:8080", Alive: true}
fmt.Println(b.URL)   // http://localhost:8080
fmt.Println(b.Alive) // true
```

Fields are accessed with a dot. Go initializes any field you don't set to its zero value: `""` for strings, `false` for bools, `0` for numbers.

## Attaching Behavior with Methods

A **method** is a function with a special receiver argument that binds it to a type. You write the receiver between `func` and the method name:

```go
func (b Backend) String() string {
    status := "down"
    if b.Alive {
        status = "up"
    }
    return fmt.Sprintf("%s [%s]", b.URL, status)
}
```

Now you can call `b.String()` on any `Backend` value. This is how Go attaches behavior to data — there's no class keyword, just a function with a receiver.

## Value Receivers vs Pointer Receivers

The receiver can be a value (`b Backend`) or a pointer (`b *Backend`). This distinction matters:

- A **value receiver** gets a copy. Changes inside the method don't affect the original.
- A **pointer receiver** gets the address. Changes inside the method modify the original.

```go
// Value receiver — b is a copy
func (b Backend) IsUp() bool {
    return b.Alive
}

// Pointer receiver — b points to the original
func (b *Backend) MarkDown() {
    b.Alive = false
}
```

Use a pointer receiver whenever you need to mutate the struct, or when the struct is large enough that copying it would be wasteful.

> **Worked Example**
>
> Here is the `Backend` struct as it will look in our load balancer — stripped down to just the fields we can define today:
>
> ```go
> package main
>
> import "fmt"
>
> type Backend struct {
>     URL   string
>     Alive bool
> }
>
> func (b *Backend) SetAlive(alive bool) {
>     b.Alive = alive
> }
>
> func (b Backend) String() string {
>     status := "down"
>     if b.Alive {
>         status = "up"
>     }
>     return fmt.Sprintf("%s [%s]", b.URL, status)
> }
>
> func main() {
>     b := Backend{URL: "http://localhost:8080", Alive: true}
>     fmt.Println(b)       // http://localhost:8080 [up]
>     b.SetAlive(false)
>     fmt.Println(b)       // http://localhost:8080 [down]
> }
> ```
>
> Notice `SetAlive` uses a pointer receiver (`*Backend`) because it modifies the struct. `String()` uses a value receiver because it only reads. When you print a struct, Go automatically calls `String()` if it's defined — that's the `fmt.Stringer` interface working behind the scenes.

## What's Next

Right now our `Backend` stores its URL as a plain string. In the next section we'll upgrade it to `*url.URL` — a rich type from the standard library — and learn why it has to be a pointer.
