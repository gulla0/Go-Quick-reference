# Go Quick Reference

This quick reference covers common Go syntax, types, collections, and basic concurrency patterns.

## 1. File / Program Basics

```go
package main // executable program (requires main())

import (
    "fmt"  // printing
    "time" // timing, Sleep, etc.
)

func main() {
    fmt.Println("Hello")
}
```

- **package main**: this code builds a runnable binary.
- **main()**: entry point; when it returns, the program exits (and all goroutines die).

---

## 2. Variables, Types & Declarations

### 2.1 Short vs explicit declaration

```go
x := 10        // declare + assign, type inferred (int)
name := "go"  // string

var y int      // declare only, zero value (0)
y = 20

var z int = 30 // explicit type + value (usually use := instead)
```

Rule of thumb:
- Use `:=` inside functions almost always.
- Use `var` when:
  - you want the zero value,
  - you declare without assigning yet,
  - you're outside a function (package level), or
  - you want to be explicit about the type.

### 2.2 Primitive numeric types (refresher)

- Signed (can be negative): `int`, `int8`, `int16`, `int32`, `int64`.
- Unsigned (only ≥ 0): `uint`, `uint8` (`byte`), `uint16`, `uint32`, `uint64`.

Signed types use one bit for the sign, so for the same bit-width they have a smaller positive range.

Bit size vs bytes:
- `int8`  → 8 bits  → 1 byte
- `int16` → 16 bits → 2 bytes
- `int32` → 32 bits → 4 bytes
- `int64` → 64 bits → 8 bytes

### 2.3 Other core types

- `float32`, `float64` – standard floats (use `float64` unless you have a reason not to).
- `bool` – `true` / `false`.
- `string` – UTF-8 text, immutable.
- `complex64` / `complex128` – complex numbers:

```go
c := 3 + 4i // real=3, imag=4
```

⸻

## 3. Collections & Structs

### 3.1 Arrays (fixed size)

```go
var a [3]int = [3]int{1, 2, 3}
```

### 3.2 Slices (dynamic — most used)

```go
nums := []int{1, 2, 3}
nums = append(nums, 4)
```

Slices are resizable views over arrays. Use slices in most cases (`[]T`).

### 3.3 Maps (key → value)

```go
m := map[string]int{
    "age":   25,
    "score": 99,
}

v := m["age"]
m["score"] = 100
```

Accessing a missing key returns the zero value for the value type; use the comma-ok idiom to test presence:

```go
v, ok := m["unknown"]
if !ok {
    // key not present
}
```

### 3.4 Structs

```go
type Person struct {
    Name string
    Age  int
}

p := Person{Name: "Harsha", Age: 30}

// method with value receiver
func (p Person) Greet() {
    fmt.Println("Hi, I'm", p.Name)
}
```

Use value receivers for immutable or small structs; use pointer receivers when a method modifies the receiver or to avoid copying large structs.

⸻

## 4. Functions & Function Types

### 4.1 Normal function

```go
func addOne(x int) int {
    return x + 1
}
```

### 4.2 Functions as values

Functions are first-class values — they can be assigned to variables, passed as arguments, and returned from other functions.

```go
// define a function type
type MathFunc func(int) int

func addOne(x int) int  { return x + 1 }
func square(x int) int  { return x * x }

func main() {
    // type inferred
    f := addOne
    fmt.Println(f(10)) // 11

    // explicit typed variable
    var g MathFunc = square
    fmt.Println(g(10)) // 100
}
```

Key distinctions:
- Function name (`addOne`) is a symbol that refers to the function implementation.
- A variable holding a function (e.g. `f`) is a value you can pass around, replace, or call.

Explicit form:

```go
var f func(int) int = addOne
// is effectively the same as
f := addOne
```

⸻

## 5. Concurrency — Goroutines & Channels

### 5.1 Goroutines

A goroutine is a lightweight concurrent execution unit.

```go
go doWork() // runs doWork() in the background

go func() {
    fmt.Println("inline goroutine")
}()
```

- The `main` goroutine controls program lifetime.
- When `main()` returns, the program exits and other goroutines are terminated.

For short demos you might see `time.Sleep(...)` to keep the program alive; prefer `sync.WaitGroup` or channels in production code.

### 5.2 Channels — mental model

Channels are typed pipes for passing values between goroutines safely.

```go
ch := make(chan int)      // unbuffered
ch2 := make(chan int, 5)  // buffered (capacity 5)

// send
ch <- 1

// receive
v := <-ch
```

- Unbuffered channels block the sender until a receiver is ready, and vice versa.
- Buffered channels allow up to N queued sends before senders block.

### 5.3 One-way channel types

You can restrict channel direction in function signatures:

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    // jobs: receive-only
    // results: send-only
}
```

### 5.4 Reading until a channel is closed

```go
for j := range jobs {
    // runs once per value sent into jobs
    // stops when channel is closed AND emptied
}
```

Notes on closing channels:
- `close(ch)` signals "no more values will be sent"; it does not remove buffered values.
- Receivers can still drain remaining values; `for range` ends once the channel is closed and empty.

⸻

## 6. Worker Pattern — What to Remember

Classic worker pool pattern:

```go
jobs := make(chan int, 5)
results := make(chan int, 5)

// start N workers
for w := 1; w <= 3; w++ {
    go worker(w, jobs, results)
}

// send jobs
for j := 1; j <= 5; j++ {
    jobs <- j
}
close(jobs)

// collect results
for a := 1; a <= 5; a++ {
    fmt.Println("Result:", <-results)
}
```

Key points:
- Multiple workers read from the same `jobs` channel; each job is delivered to exactly one worker.
- Fast workers can process more jobs; slow ones fewer.
- Close `jobs` to signal no more work — workers will exit once the channel is drained.
- Reading more times from `results` than values produced will block; ensure balanced sends/receives.

⸻

## 7. Quick "When to Use What" Reminders

- `:=` — Use inside functions when you have a value immediately.
- `var` — Use for zero values, late assignment, or package-level declarations.
- Slice vs Array — Use slices (`[]T`) ~99% of the time; arrays (`[N]T`) are for low-level or fixed-size needs.
- Map vs Struct — Use structs for fixed, typed fields; use maps for dynamic key sets.
- Goroutine — Use for concurrent independent work (IO, network, many independent tasks).
- Channel — Use for communication or coordination between goroutines instead of shared-memory locks.

