# Proposal 002: Pointer Semantics — How Bak Handles Go's `*T` and `&`

**Status:** Proposed
**Date:** 2026-02-28

---

## The Problem

Go uses pointers (`*T`, `&x`) for three distinct purposes:

| Purpose              | Go syntax                          | Example                                       |
|----------------------|------------------------------------|------------------------------------------------|
| **Nilability**       | `*T` can be `nil`                  | `var u *User = nil`                            |
| **Mutability**       | pointer receiver = can modify      | `func (u *User) SetName(n string)`             |
| **Performance**      | avoid copying large structs        | `func Process(u *User)` instead of `User`      |

Bak already has features that cover two of these:
- **Nilability** → `T?` (null safety system)
- **Mutability** → `val`/`var` distinction

The question: does Bak expose pointer syntax, hide it, or replace it with something else?
And crucially: how does this interact with **100% bidirectional Go interop**?

---

## Go Interop Constraints

Any solution MUST handle these real-world Go patterns:

```go
// 1. Functions that require pointer arguments
func json.Unmarshal(data []byte, v any) error     // v must be a pointer
func fmt.Sscan(str string, a ...any) (n int, err error)

// 2. Struct fields that are pointers
type http.Request struct {
    URL    *url.URL
    Body   io.ReadCloser
    TLS    *tls.ConnectionState  // nil if not TLS
}

// 3. Pointer receivers (mutation)
func (s *Server) ListenAndServe() error

// 4. Pointer-to-pointer (rare but exists)
func (**Node) Next() *Node

// 5. Interface satisfaction requiring pointer receiver
type Stringer interface { String() string }
// (*MyType).String() satisfies Stringer, but MyType.String() does not

// 6. Returning pointers (constructor pattern)
func NewUser(name string) *User { return &User{Name: name} }
```

---

## Option P1: Hide Pointers Entirely (Compiler Decides)

### Design

No `*` or `&` in Bak syntax. The compiler determines value vs reference semantics
based on context. `T?` is the only way to express "might not exist."

```kotlin
// Values and nullability — no pointer syntax
val user = User("Alice", 30)
val maybe: User? = FindUser(42)       // T? → *T in Go

// Functions — compiler decides pass-by-value vs pass-by-reference
fun UpdateAge(user: User, newAge: Int): User {
    return user.copy(age = newAge)     // immutable style
}

// Mutation via method — compiler generates pointer receiver
fun User.incrementAge() {
    this.age += 1                      // compiler sees mutation → *User receiver
}

// Go interop — compiler auto-adds & where Go expects pointer
import "encoding/json"
val user = User("Alice", 30)
json.Unmarshal(data, user)             // compiler: json.Unmarshal(data, &user)

// Struct fields that are pointers in Go
val req: http.Request = getRequest()
val url: URL? = req.URL                // Go's *url.URL mapped to URL?
val tls: TLS? = req.TLS               // Go's *tls.ConnectionState → TLS?

// Constructor pattern — Go returns *User, Bak sees User (non-null)
val user = NewUser("Alice")            // Go: *User → Bak: User
val user2: User? = MaybeNewUser("?")   // Go: *User → Bak: User? (if can be nil)
```

```go
// Generated Go
user := User{Name: "Alice", Age: 30}

// Compiler-generated pointer receiver
func (u *User) IncrementAge() {
    u.Age += 1
}

// Compiler auto-inserts &
json.Unmarshal(data, &user)
```

### How Interop Rules Work

| Go signature                     | Bak sees                              | Rule                                    |
|----------------------------------|---------------------------------------|-----------------------------------------|
| `func F(x *T)`                   | `fun F(x: T)`                         | Compiler auto-adds `&` at call site     |
| `func F(x *T)` where x can be nil| `fun F(x: T?)`                        | Nullable = pointer                      |
| `func F() *T`                    | `fun F(): T` or `fun F(): T?`         | Depends on whether Go func returns nil  |
| `*T` struct field                | `val field: T?`                       | Pointer field = nullable                |
| `func (t *T) Method()`           | `fun T.Method()` with mutation        | Compiler uses pointer receiver          |
| `**T`                            | ???                                   | **UNSUPPORTED** — need escape hatch     |

### Pros

| Advantage                                                     |
|---------------------------------------------------------------|
| Cleanest possible syntax — zero pointer noise                 |
| Null safety (`T?`) perfectly subsumes nilability              |
| Beginners never need to think about pointers                  |
| Compiler can optimize value vs reference based on struct size |
| Closest to Kotlin/Java mental model                           |
| `val`/`var` + `T?` cover 95% of pointer use cases            |

### Cons

| Disadvantage                                                    |
|-----------------------------------------------------------------|
| Loss of explicit control over memory layout                     |
| Go functions that REQUIRE `*T` need compiler magic              |
| `**T` (pointer-to-pointer) is impossible — exists in some Go code |
| Performance-sensitive code may need manual control              |
| Compiler must analyze Go function signatures to know when to add `&` |
| Implicit behavior can surprise: "why is this a pointer in Go?" |
| How to distinguish `func F(x *T)` "must be non-nil pointer" vs "nullable pointer"? |

### Open Questions

1. How does the compiler know if a Go `*T` parameter can be nil or not?
2. What happens with `**T` in Go APIs? (rare but exists in tree structures)
3. How does the user force a specific memory layout for performance?
4. How to handle Go's `unsafe.Pointer`?

---

## Option P2: `ref` Keyword for Reference Types

### Design

Introduce a `ref` keyword that maps directly to Go's `*T`. The default is value type.
`ref` is explicit and controlled.

```kotlin
// Value types (default)
val user = User("Alice", 30)           // Go: User{...}
var user2 = User("Bob", 25)            // Go: User{...} (mutable binding)

// Reference types (explicit)
ref user3 = User("Charlie", 35)        // Go: &User{...}

// Nullable reference
ref user4: User? = null                // Go: var user4 *User = nil

// ref in function parameters
fun Process(user: ref User) {          // Go: func Process(user *User)
    user.name = "modified"             // mutation through reference
}

// ref receiver (pointer receiver in Go)
fun User.incrementAge() ref {          // Go: func (u *User) IncrementAge()
    this.age += 1
}

// ref in data class fields
data class TreeNode(
    val value: Int,
    val left: ref TreeNode?,           // Go: *TreeNode (can be nil)
    val right: ref TreeNode?           // Go: *TreeNode (can be nil)
)

// Go interop — direct mapping
import "encoding/json"
ref user = User("Alice", 30)
json.Unmarshal(data, user)             // user is already a ref, no & needed

// or: auto-ref at call site
val user = User("Alice", 30)
json.Unmarshal(data, ref user)         // explicit: "pass as reference"

// Constructor pattern — Go returns *User
val user = NewUser("Alice")            // returns ref User, auto-deref to User
ref user2 = NewUser("Bob")             // keep as reference

// Pointer-to-pointer (rare)
ref ref node: TreeNode? = ...          // Go: **TreeNode
```

```go
// Generated Go
user := User{Name: "Alice", Age: 30}       // val
user3 := &User{Name: "Charlie", Age: 35}   // ref

func Process(user *User) { ... }            // ref param
func (u *User) IncrementAge() { ... }       // ref receiver
```

### Pros

| Advantage                                                     |
|---------------------------------------------------------------|
| Explicit control over value vs reference semantics            |
| Direct mapping to Go's `*T` — no compiler magic              |
| `ref` is self-documenting: "this is a reference"              |
| Handles `**T` via `ref ref`                                   |
| Performance control: developer chooses when to use references |
| Go interop is transparent — you see exactly what generates    |
| No ambiguity about what the compiler does                     |

### Cons

| Disadvantage                                                    |
|-----------------------------------------------------------------|
| New keyword (`ref`) adds complexity                             |
| Three concepts to learn: `val` + `var` + `ref`                 |
| `ref` + `var` + `?` combinations could be confusing            |
| Still thinking about pointers, just with different syntax       |
| `ref ref` for `**T` is ugly                                    |
| More verbose than P1 for common cases                          |
| Bak feels less like Kotlin, more like "Go with different syntax"|

### Combination Matrix

| Declaration              | Mutable? | Reference? | Nullable? | Go equivalent         |
|--------------------------|----------|------------|-----------|------------------------|
| `val x: T`               | no       | no         | no        | `x := T{}`             |
| `var x: T`               | yes      | no         | no        | `var x T`              |
| `val x: T?`              | no       | -          | yes       | `var x *T = nil`       |
| `ref x: T`               | -        | yes        | no        | `x := &T{}`            |
| `ref x: T?`              | -        | yes        | yes       | `var x *T`             |
| `var ref x: T`           | yes      | yes        | no        | `x := &T{}`           |

---

## Option P3: Auto-Deref with Explicit `&` (Rust-Inspired)

### Design

Keep `&` for "take a reference" (rare, mostly for Go interop).
No `*` needed — auto-dereference everywhere like Rust.
`T?` covers nilability. `mut` marks mutating methods.

```kotlin
// No pointer syntax in normal code
val user = User("Alice", 30)
val maybe: User? = FindUser(42)

// & only at call sites when Go requires a pointer
import "encoding/json"
json.Unmarshal(data, &user)            // explicit: "pass address"

// Mutating methods use 'mut' keyword
fun User.incrementAge() mut {
    this.age += 1                      // Go: pointer receiver
}

// Calling a mutating method
var user = User("Alice", 30)
user.incrementAge()                    // OK — user is var
// val user2 = User("Bob", 25)
// user2.incrementAge()                // COMPILE ERROR — user2 is val

// Fields — auto-deref
val req: http.Request = getRequest()
val host = req.URL.Host                // auto-deref: Go's req.URL is *url.URL
                                       // but Bak accesses it like a value

// Nullable = pointer that can be nil
val tls: TLSState? = req.TLS          // Go's *tls.ConnectionState

// Pointer-to-pointer — use & twice
val node: Node? = ...
funcThatWantsDoublePointer(&&node)     // ugly but possible
```

```go
// Generated Go
json.Unmarshal(data, &user)

func (u *User) IncrementAge() {
    u.Age += 1
}

// Auto-deref doesn't change generated code
host := req.URL.Host   // same in Go
```

### Pros

| Advantage                                                     |
|---------------------------------------------------------------|
| Minimal syntax addition: only `&` at call sites               |
| Auto-deref eliminates `*` noise everywhere                    |
| `mut` is clear about intent: "this modifies the receiver"     |
| Familiar to Rust developers                                   |
| `&` is explicit at the point where you need it                |
| Compiler can still optimize value vs reference in pure Bak    |

### Cons

| Disadvantage                                                    |
|-----------------------------------------------------------------|
| `&` is still "pointer thinking" — just half the syntax          |
| `mut` keyword: is it a receiver annotation? A method modifier?  |
| Two mutability concepts: `var` (binding) and `mut` (method)     |
| `&&` for double pointer is ugly                                 |
| Mixing Kotlin conventions (`val`/`var`) with Rust (`mut`, `&`)  |
| Need to explain when `&` is needed vs when compiler handles it  |
| `mut` on methods doesn't cover all mutation cases (e.g., passing to Go func that mutates) |

---

## Option P4: Smart Defaults + `ptr` Escape Hatch

### Design

99% of Bak code has no pointer concept. The compiler handles everything.
For the 1% that needs explicit control (Go interop, performance), a `ptr<T>`
type and `&` operator provide an escape hatch.

```kotlin
// Normal Bak: no pointers
val user = User("Alice", 30)
val maybe: User? = FindUser(42)

// Mutation through method annotation
fun User.incrementAge() mutating {
    this.age += 1                      // compiler: pointer receiver
}

// --- Escape hatch for Go interop ---

// & at call site when Go needs a pointer
import "encoding/json"
json.Unmarshal(data, &user)

// ptr<T> type for when you need to store/pass explicit pointers
val p: ptr<User> = &user              // explicit pointer
val p2: ptr<User>? = null             // nullable pointer

// Dereferencing — automatic in most cases
println(p.name)                        // auto-deref
val raw: User = *p                     // explicit deref (if needed)

// ptr in function signatures (for Go interop wrappers)
fun Decode(data: ByteArray, target: ptr<User>): Result<Unit, Error> {
    return json.Unmarshal(data, target)
}

// ptr-to-ptr (rare)
val pp: ptr<ptr<Node>> = ...

// Struct fields that are pointers in Go
data class TreeNode(
    val value: Int,
    val left: ptr<TreeNode>?,          // explicit: this IS a pointer in Go
    val right: ptr<TreeNode>?
)

// Most Go interop doesn't need ptr — compiler handles it:
val resp = http.Get("https://example.com")  // returns *http.Response → Bak: Response
val body = resp.Body                         // auto-deref
```

```go
// Generated Go — normal code
user := User{Name: "Alice", Age: 30}

// Escape hatch generates explicit pointer code
p := &user
json.Unmarshal(data, &user)

// ptr<T> fields
type TreeNode struct {
    Value int
    Left  *TreeNode
    Right *TreeNode
}
```

### Pros

| Advantage                                                     |
|---------------------------------------------------------------|
| Clean by default — 99% of code has no pointer syntax          |
| `ptr<T>` is self-documenting and type-safe                    |
| `&` only appears at call sites where actually needed          |
| Full control available when needed (perf, interop)            |
| Auto-deref means `ptr<T>` is mostly transparent               |
| `mutating` is Swift-inspired, clear intent                    |
| Handles `**T` via `ptr<ptr<T>>`                               |
| Clear separation: "Bak world" vs "Go interop world"          |

### Cons

| Disadvantage                                                    |
|-----------------------------------------------------------------|
| Two "worlds": pointer-free Bak and pointer-aware interop        |
| `ptr<T>` is un-Kotlin-like syntax                               |
| `&` and `*` still exist in the escape hatch                     |
| Developers must learn when the escape hatch is needed           |
| `mutating` is another keyword to learn                          |
| Compiler magic for "normal" code may surprise                   |
| `ptr<ptr<T>>` is verbose for double pointers                    |

---

## Comparison Matrix

| Feature                         | P1 (hide)    | P2 (ref)     | P3 (auto-deref) | P4 (smart+escape) |
|---------------------------------|:------------:|:------------:|:----------------:|:------------------:|
| No pointer syntax by default    | yes          | no           | mostly           | yes                |
| Explicit pointer control        | no           | yes          | partial (`&`)    | yes (`ptr<T>`)     |
| `T?` covers nilability          | yes          | yes          | yes              | yes                |
| `**T` support                   | no           | `ref ref`    | `&&`             | `ptr<ptr<T>>`      |
| Go interop transparency         | low (magic)  | high         | medium           | medium-high        |
| New keywords                    | none         | `ref`        | `mut`            | `mutating`         |
| New syntax                      | none         | `ref x`      | `&x`             | `ptr<T>`, `&x`     |
| Kotlin familiarity              | highest      | medium       | low (Rust-ish)   | high               |
| Performance control             | none         | full         | partial          | full               |
| Compiler complexity             | highest      | lowest       | medium           | medium             |
| Beginner friendliness           | highest      | medium       | medium           | high               |
| "Escape hatch" needed           | always       | never        | sometimes        | rarely             |

---

## Interaction with Other Features

### With Null Safety (`T?`)

| Pointer option | `T?` means                          | Go mapping              |
|----------------|-------------------------------------|--------------------------|
| P1 (hide)      | `T?` = might be nil (was `*T`)      | `*T` always              |
| P2 (ref)       | `T?` = nullable value, `ref T?` = nullable reference | depends |
| P3 (auto-deref)| `T?` = might be nil                 | `*T` for nullable        |
| P4 (escape)    | `T?` = might be nil, `ptr<T>?` = nullable pointer | depends |

### With Option<T>

`Option<T>` is always a value type — no pointer semantics. It wraps `T`, not `*T`.
The bridge `T?.toOption()` works regardless of pointer strategy.

### With Result<T, E>

No interaction — `Result` is a value type. Error handling is orthogonal to pointers.

### With Data Classes

Data classes are value types by default. In P2, `ref` makes them reference types.
In P1/P3/P4, the compiler decides based on size and usage.

---

## Recommendation

**No recommendation yet.** This decision requires weighing:

1. How much Go interop friction is acceptable?
2. How important is explicit performance control?
3. How Kotlin-like should Bak feel?
4. What percentage of Bak code will need pointer awareness?

The answer depends on the primary audience:
- **Go developers wanting nicer syntax** → P2 (ref) or P4 (escape hatch)
- **Kotlin developers wanting Go's performance** → P1 (hide) or P4 (escape hatch)
- **Rust developers wanting Go's ecosystem** → P3 (auto-deref)

P4 appears to be the best compromise, but P1's simplicity is tempting if the
compiler can be smart enough.
