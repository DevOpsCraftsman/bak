# Proposal 003: Mutability, Declarations, and Syntax Identity

**Status:** Proposed
**Date:** 2026-02-28

---

## Why This Is Foundational

Every line of Bak code starts with a declaration or a function signature.
The syntax chosen here defines:
- How Bak **feels** to write (Kotlin? Rust? Go? something new?)
- How **token-efficient** the language is for AI-assisted coding
- How **trivially** Bak compiles to Go
- How Bak brings **immutability** without alienating Go developers

This proposal also resolves Proposal 002 (pointer semantics) and establishes
Bak's overall syntax identity.

---

## Design Constraints

| Constraint                      | Why                                                      |
|---------------------------------|----------------------------------------------------------|
| Immutable by default            | Safety, predictability, FP — core Bak promise            |
| Token-efficient                 | AI-assisted coding — fewer tokens = faster, cheaper      |
| Trivial Go codegen              | `:=` should map to `:=`, not require complex transforms  |
| Go developers not alienated     | Familiar syntax, not a foreign language                  |
| No borrow checker               | Go's GC handles memory — don't add Rust's complexity     |
| `&` maps to `*T` in Go          | 100% interop requires explicit pointer control           |
| One idiomatic way               | Avoid parallel syntaxes that mean the same thing         |

---

## Cross-Cutting Decision: Syntax Style

Before choosing a mutability model, we must decide the **surface syntax**.
Two axes: type annotation style, and function keyword.

### Type Annotations: `:` or no `:`?

```
// Kotlin-style (colon separator)
fn add(a: Int, b: Int): Int
x: Int := 42                   // `: ... :=` — redundant, ugly

// Go-style (no colon)
fn add(a Int, b Int) Int
var x Int = 42                  // reads like Go
```

The `:` is a Kotlin/TypeScript convention. Go doesn't use it. Arguments:

| For `:` (Kotlin-style)                          | Against `:` (Go-style)                           |
|-------------------------------------------------|--------------------------------------------------|
| Visual separator between name and type           | Extra character on every param — adds up          |
| Familiar to Kotlin/TypeScript/Rust developers    | `x: Int := 42` has both `:` and `:=` — redundant |
| Used by most modern languages                    | Go devs read `a int` fine for 15 years            |
|                                                  | Token cost on every function signature            |
|                                                  | If Bak is Go-first, colon is foreign              |

**Recommendation: Go-style (no colon).** If Bak uses `:=` and `var` from Go,
then the colon is an unnecessary Kotlin import. Consistency wins.

### Function Keyword: `fun` or `fn`?

| Keyword | Characters | Used by                        |
|---------|:----------:|--------------------------------|
| `fun`   | 3          | Kotlin                         |
| `fn`    | 2          | Rust, Zig                      |
| `func`  | 4          | Go                             |

`fn` saves 1 character over `fun` and 2 over `func`, on every function declaration.
In a typical file with 10-20 functions, that's 10-40 tokens saved.

**Recommendation: `fn`.** Shorter, used by Rust (which Bak borrows from),
and distinct from both Go (`func`) and Kotlin (`fun`).

### Combined Syntax Identity

These two decisions position Bak's visual identity:

```go
// Go
func add(a int, b int) int {
    return a + b
}

// Kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}

// Bak (proposed)
fn add(a Int, b Int) Int {
    return a + b
}
```

Bak reads like **Go with shorter keywords and immutability**.
Not like Kotlin with Go compilation.

---

## Option M1: `:=` Immutable by Default + `var` for Mutability

### Core Idea

Take Go's most iconic syntax (`:=`) and **invert its default**: immutable.
Use Go's own `var` keyword — repurposed to mean "this binding is mutable."

### Variable Declarations

```go
// Immutable — the common case, zero keyword overhead
x := 42
name := "Alice"
user := User("Alice", 30)

// Mutable — explicit opt-in with var
var count := 0
count = 1              // OK — count is var
count = count + 1      // OK

// With explicit type (when inference isn't enough)
var x Int = 42
var count Int = 0
```

### Go Codegen

```go
// x := 42             →  x := 42           (identical!)
// name := "Alice"     →  name := "Alice"
// var count := 0      →  count := 0         (Go has no immutability — compile-time only)
// var x Int = 42      →  var x int = 42     (identical!)
```

The immutability guarantee is **compile-time only** in the Bak compiler.
Go's generated code is identical — the safety comes from Bak, not Go.

### Function Parameters

```go
// By value (copy) — immutable, the default
fn process(data User) {
    // data.name = "Bob"     // COMPILE ERROR — params are immutable
    println(data.name)       // OK — read-only
}

// By reference (pointer) — for mutation or large structs
fn update(data &User) {
    data.name = "Bob"        // OK — & means pointer, mutation allowed
}

// Multiple params
fn transfer(from &Account, to &Account, amount Int) {
    from.balance -= amount
    to.balance += amount
}
```

```go
// fn process(data User)            →  func process(data User)
// fn update(data &User)            →  func update(data *User)
// fn transfer(from, to, amount)    →  func transfer(from *Account, to *Account, amount int)
```

### Why Not `mut` in Params?

In Go, if you have a `*T`, you can **always** modify through it.
There is no "read-only pointer" in Go. No borrow checker. No `&T` vs `&mut T`.

Bak mirrors this: `&User` = pointer = can modify. Period.
This keeps the mental model simple and the codegen trivial.

**Two concepts, each in its own context:**

| Context               | Concept | Meaning                          |
|-----------------------|---------|----------------------------------|
| Variable declarations | `var`   | "This binding can be reassigned" |
| Function parameters   | `&`     | "This is a pointer to the original" |

No `mut`. No `&mut`. No combinations to memorize.

### Method Receivers

```go
// Value receiver — can't modify the struct
fn User.fullName() String {
    return "${this.firstName} ${this.lastName}"
}

// Pointer receiver — can modify the struct
fn (&User).incrementAge() {
    this.age += 1
}
```

```go
// fn User.fullName() String          →  func (u User) FullName() string
// fn (&User).incrementAge()          →  func (u *User) IncrementAge()
```

### Struct Fields

```go
data class User(
    name String,           // immutable field (default)
    var age Int             // mutable field
)

user := User("Alice", 30)
// user.name = "Bob"       // COMPILE ERROR — name is immutable
user.age = 31              // OK — age is var
```

```go
// Both generate the same Go struct (immutability is Bak-enforced):
// type User struct {
//     Name string
//     Age  int
// }
```

### Nullable + Reference Combinations

```go
name := "Alice"                // String, non-null, immutable
name String? := null           // String?, nullable, immutable
var name := "Alice"            // String, non-null, mutable binding
var name String? := null       // String?, nullable, mutable binding

p := &user                     // &User (pointer to user)
p &User? := null               // nullable pointer (can be nil)
```

### Full Example

```go
data class Config(
    host String,
    port Int = 8080,
    var retries Int = 3
)

fn loadConfig(path String) Result<Config, Error> {
    content := os.ReadFile(path)?
    config := json.Unmarshal<Config>(content)?
    return Ok(config)
}

fn serve(config Config) {
    var attempts := 0
    router := http.NewServeMux()

    router.HandleFunc("GET /health") { w, _ ->
        attempts += 1
        w.Write("ok")
    }

    addr := ":${config.port}"
    log.Printf("Listening on %s", addr)
    http.ListenAndServe(addr, router)
}
```

### Pros

| Advantage                                                                |
|--------------------------------------------------------------------------|
| Zero keyword for the most common case (immutable) — maximum token savings |
| `:=` is Go's own syntax — zero learning curve for Go developers          |
| `var` is Go's own keyword — repurposed, not invented                     |
| Trivial codegen: Bak `:=` → Go `:=` (identical!)                        |
| `&` maps directly to Go's `*T` — no translation layer                   |
| Only 2 concepts: `var` (reassignable) and `&` (pointer)                  |
| No `mut`, no `&mut`, no `ref` — minimal keyword surface                  |
| Immutability is a compile-time guarantee — zero runtime cost             |
| No `:` in type annotations — same as Go                                  |
| `fn` is shorter than `fun` (Kotlin) and `func` (Go)                     |

### Cons

| Disadvantage                                                              |
|---------------------------------------------------------------------------|
| `:=` is mutable in Go — semantic inversion could confuse Go developers    |
| `var` means "declare with type" in Go — repurposing changes meaning       |
| No way to express "read-only reference" (but Go can't enforce it either)  |
| Pure Bak code doesn't benefit from `&` — only useful for Go interop       |

### Open Questions

1. Is inverting `:=`'s semantics a deal-breaker for Go developers?
2. Should `var` require `:=` or allow `=`? (i.e., `var x := 0` vs `var x = 0`)
3. How to handle Go's zero values? (`var x Int` → `var x int` with zero value `0`)

---

## Option M2: `:=` Immutable + `mut` for Mutability

### Core Idea

Same as M1 but use Rust's `mut` instead of Go's `var`.
This avoids the semantic overload of `var` (which means something different in Go).

### Variable Declarations

```go
// Immutable (default)
x := 42
name := "Alice"

// Mutable
mut count := 0
count = 1              // OK

// With explicit type
x Int := 42
mut count Int := 0
```

### Function Parameters & References

Same as M1 — `&` for references, no `mut` in params:

```go
fn process(data User) { ... }        // by value, immutable
fn update(data &User) { ... }        // by reference (pointer)
```

### Struct Fields

```go
data class User(
    name String,           // immutable
    mut age Int             // mutable
)
```

### Pros

| Advantage                                                              |
|------------------------------------------------------------------------|
| `mut` has no pre-existing meaning in Go — no confusion                 |
| Familiar to Rust developers                                            |
| Clear intent: `mut` = "I need to change this"                          |
| Same token count as `var` (3 chars)                                    |
| Everything else from M1 applies                                        |

### Cons

| Disadvantage                                                            |
|-------------------------------------------------------------------------|
| `mut` is not a Go keyword — feels more Rust than Go                     |
| Introduces a new keyword that Go devs must learn                        |
| `mut` + `:=` is a Rust/Go hybrid that belongs to neither language       |

---

## Option M3: `val` + `var` (Kotlin Classic)

### Core Idea

Keep Kotlin's `val`/`var` as-is. Add `&` only for pointer control.
The most conservative option — maximum Kotlin familiarity, but with Go-style
type annotations (no colon) and `fn` keyword.

### Variable Declarations

```go
val x = 42
val name = "Alice"

var count = 0
count = 1

val x Int = 42
var count Int = 0
```

### Function Parameters & References

```go
fn process(data User) { ... }        // by value, immutable
fn update(data &User) { ... }        // by reference (pointer)
```

### Struct Fields

```go
data class User(
    val name String,
    var age Int
)
```

### Pros

| Advantage                                                              |
|------------------------------------------------------------------------|
| Familiar to Kotlin developers — zero learning curve                    |
| `val`/`var` is well-understood (Kotlin, Scala, Swift)                  |
| `val` reads as "value" — self-documenting                              |

### Cons

| Disadvantage                                                            |
|-------------------------------------------------------------------------|
| `val ` is 4 chars on EVERY immutable declaration — token overhead       |
| Not familiar to Go developers — feels like a different language          |
| `var` in Bak (mutable binding) vs `var` in Go (type declaration) clash  |
| Less innovative — "it's just Kotlin syntax with Go types"               |
| Token overhead: `val x = 42` (10 chars) vs `x := 42` (7 chars)         |

---

## Option M4: `:=` Immutable + `var` Go-True (Recommended)

### Core Idea

Keep `var` with its **actual Go meaning** — "declare with type,
may not be initialized yet" — and use `:=` for initialized declarations (immutable).
`var` declarations are mutable by nature (same as Go).

This is the only option where `var` means **exactly** the same thing as in Go.

### Variable Declarations

```go
// Initialized (immutable by default) — type inferred
x := 42
name := "Alice"

// Uninitialized — needs explicit type (like Go's var)
var count Int                // zero value: 0
var name String              // zero value: ""
var user User?               // zero value: null

// Uninitialized + assigned later
var count Int
count = computeCount()
count = 99                   // OK — var is always mutable

// Mutable initialized — var + :=
var count := 0               // initialized AND mutable
count = 1                    // OK
```

### Function Parameters & References

```go
fn process(data User) { ... }         // by value
fn update(data &User) { ... }         // by reference

fn add(a Int, b Int) Int {
    return a + b
}

fn readFile(path String) Result<[]byte, Error> {
    return os.ReadFile(path)
}
```

### Method Receivers

```go
fn User.fullName() String {
    return "${this.firstName} ${this.lastName}"
}

fn (&User).incrementAge() {
    this.age += 1
}
```

### Struct Fields

```go
data class User(
    name String,               // immutable (default, no keyword)
    var age Int                 // mutable
)
```

### Go Codegen

```go
// x := 42              →  x := 42            (identical)
// var count Int         →  var count int       (identical!)
// var count := 0        →  count := 0
// fn add(a Int, b Int)  →  func add(a int, b int)
```

### Pros

| Advantage                                                              |
|------------------------------------------------------------------------|
| `var` means the SAME THING as in Go — zero confusion for Go devs       |
| `:=` keeps its Go feel — just immutable by default                     |
| `var` for uninitialized maps perfectly to Go's zero values             |
| Covers a real use case: "I need a variable, I'll fill it later"        |
| Minimal keyword invention — reuses Go's existing vocabulary            |
| `fn` shorter than `func` (Go) and `fun` (Kotlin)                      |
| No `:` in type annotations — same visual as Go                        |
| `&` maps directly to `*T` — trivial codegen                           |

### Cons

| Disadvantage                                                            |
|-------------------------------------------------------------------------|
| `var` has two roles: uninitialized AND mutable — could be confusing     |
| `var count := 0` looks redundant — "why not just `count := 0`?"        |
| Two forms for initialized mutable: `var x := 0` and `var x Int = 0`    |

### Full Example

```go
import "net/http"
import "encoding/json"

data class Todo(
    id String,
    title String,
    var completed Bool = false
)

sealed class TodoError {
    data class NotFound(id String)
    data class Invalid(message String)
}

fn createTodo(title String) Result<Todo, TodoError> {
    if title.isBlank() {
        return Err(TodoError.Invalid("title required"))
    }
    todo := Todo(id = generateId(), title = title.trim())
    return Ok(todo)
}

fn handleCreate(w &http.ResponseWriter, r &http.Request) {
    body := io.ReadAll(r.Body)?
    title := string(body)

    match createTodo(title) {
        is Ok(todo) -> {
            w.WriteHeader(201)
            json.NewEncoder(w).Encode(todo)
        }
        is Err(e) -> match e {
            is TodoError.Invalid(msg) -> {
                w.WriteHeader(400)
                w.Write(msg)
            }
            is TodoError.NotFound(id) -> {
                w.WriteHeader(404)
                w.Write("not found: $id")
            }
        }
    }
}

fn main() {
    mux := http.NewServeMux()
    mux.HandleFunc("POST /todos", handleCreate)

    var attempts := 0
    go {
        attempts += 1
        log.Println("Starting server on :8080")
        http.ListenAndServe(":8080", mux)
    }

    signal := make(chan os.Signal, 1)
    <-signal
}
```

---

## Comparison Matrix

| Aspect                     | M1 (`:=`+`var`)     | M2 (`:=`+`mut`)     | M3 (`val`+`var`)     | M4 (`:=`+`var` Go-true) |
|----------------------------|:--------------------:|:--------------------:|:--------------------:|:------------------------:|
| **Tokens (immutable)**     | 2 (`:=`)             | 2 (`:=`)             | 4 (`val `)           | 2 (`:=`)                 |
| **Tokens (mutable)**       | 6 (`var :=`)         | 6 (`mut :=`)         | 6 (`var =`)          | 6 (`var :=`)             |
| **Go dev familiarity**     | high                 | medium               | low                  | highest                  |
| **Kotlin dev familiarity** | low                  | low                  | highest              | medium                   |
| **Rust dev familiarity**   | low                  | high                 | low                  | low                      |
| **`var` confusion risk**   | medium (repurposed)  | none (no `var`)      | medium (≠ Go `var`)  | none (same as Go)        |
| **Codegen simplicity**     | trivial              | trivial              | simple               | trivial                  |
| **New keywords**           | none                 | 1 (`mut`)            | 1 (`val`)            | none                     |
| **Zero values**            | no                   | no                   | no                   | yes                      |
| **Innovation**             | high                 | medium               | none                 | high                     |
| **`:` in types**           | no                   | no                   | no                   | no                       |
| **Function keyword**       | `fn`                 | `fn`                 | `fn`                 | `fn`                     |

---

## Interaction with Other Proposals

### Proposal 002 (Pointer Semantics) — RESOLVED

This proposal resolves Proposal 002. The answer across all options:
- `&T` in parameters and types = Go's `*T` (pointer)
- `&x` at call sites = Go's `&x` (take address)
- No `*` dereferencing — auto-deref everywhere
- `&T?` for nullable pointers
- No `&mut` — Go doesn't distinguish read-only vs mutable pointers

### Proposal 001 (Pattern Matching)

No interaction. `match`/`when` work identically with all options.

### Null Safety (T?)

No interaction. `T?` works the same regardless of declaration syntax.

### Option<T> / Result<T, E>

No interaction. These are value types unaffected by declaration syntax.

---

## Recommendation

**M4 (`:=` immutable + `var` Go-true)** offers the best balance:

1. **Go developers**: `var` and `:=` mean exactly what they expect
2. **Token efficiency**: `:=` for the common case (0 keyword), `fn` (2 chars)
3. **No colon**: type annotations like Go, not Kotlin
4. **Immutability**: default, no overhead, compile-time only
5. **Zero values**: `var x Int` gives Go's zero value — a real use case
6. **Codegen**: both `:=` and `var` map 1:1 to Go
7. **No new keywords**: everything reused from Go
8. **`&` for pointers**: direct mapping to `*T`, no `mut`/`ref`/`&mut`

---

## Summary: What Bak Adds to Go

| Go                           | Bak                           | What changed                  |
|------------------------------|-------------------------------|-------------------------------|
| `x := 42` (mutable)         | `x := 42` (immutable)        | Default flipped to safety     |
| `var x int` (uninitialized)  | `var x Int` (same!)           | Same concept, same keyword    |
| — (no mutable initialized)  | `var x := 42` (mutable init) | Explicit mutable opt-in       |
| `*T` / `&x`                 | `&T` / `&x`                  | `&` replaces `*` in types     |
| `func f(x int)`             | `fn f(x Int)`                 | `fn` shorter, types uppercase |
| `func f(x: int): int`       | `fn f(x Int) Int`             | No colon — same as Go         |
| — (everything mutable)      | immutable by default          | Bak's core safety guarantee   |

Bak doesn't fight Go. It adds a **safety layer on top of Go's own syntax**.
