# Bak Language — Design Discussion Record

**Date:** 2026-02-27
**Participants:** Alexandre Poitevin, Claude (AI pair)

This document records the full design discussion that led to the Bak language specification.
Every decision point, the options considered, and the rationale for each choice are preserved
as a reference for future contributors and design revisions.

---

## 1. What is Bak?

**Vision:** A programming language that is to Go what Kotlin is to Java:
- More elegant syntax
- Null-safe by default
- Purer OOP and FP constructs
- 100% bidirectional interop with Go

**Name:** Bak

---

## 2. Compilation Target

**Question:** What is the primary compilation target?

| Option                  | Description                                                                        | Chosen? |
|-------------------------|------------------------------------------------------------------------------------|---------|
| **Compile to Go source** | Transpile `.bak` → `.go` files, then `go build`. Max interop, fastest path.        | YES     |
| Compile to Go AST       | Generate Go AST nodes directly via `go/ast`, tighter coupling to Go internals.     | No      |
| Compile to LLVM IR       | Independent backend like Rust. Maximum freedom, but Go interop via FFI only.       | No      |
| Hybrid approach          | Start with Go source transpilation, evolve toward native backend later.            | No      |

**Decision:** Compile to Go source. This mirrors Kotlin's approach (compile to JVM bytecode) and
gives the strongest interop story from day one.

**Pipeline:**
```
.bak source → [Bak Compiler] → .go source → [go build] → native binary
```

---

## 3. Compiler Implementation Language

**Question:** What language should the Bak compiler itself be written in?

| Option          | Description                                                                    | Chosen? |
|-----------------|--------------------------------------------------------------------------------|---------|
| **Go**          | Dogfooding the ecosystem. Excellent parsing libraries. Fastest path.           | YES     |
| Rust            | Great for compilers. Adds second toolchain dependency.                         | No      |
| Bak (self-hosted) | Ultimate goal, but needs bootstrap compiler first. Phase 2 milestone.        | No      |
| TypeScript/Node  | Rapid prototyping, rich parser ecosystem. Unusual for Go-targeting language.  | No      |

**Decision:** Go. Staying in the ecosystem for maximum coherence. Go's `go/ast`, `go/parser`,
`go/printer` packages provide excellent tooling for generating Go source code.

---

## 4. Core Language Features (v0.1 Non-Negotiables)

**Question:** Which Kotlin-inspired features are TOP priorities?

| Feature                                  | Description                                                        | Chosen? |
|------------------------------------------|--------------------------------------------------------------------|---------|
| **Null safety (?, !!, ?:)**              | Optional types with compile-time null checks. T vs T? distinction. | YES     |
| **Data classes + pattern matching**      | Immutable value types with auto-generated methods, destructuring.  | YES     |
| **Extension functions**                  | Add methods to existing types without modifying them.              | YES     |
| **First-class FP (lambdas, HOF, val/var)** | Trailing lambdas, map/filter/reduce, expression-based syntax.    | YES     |

**Decision:** All four selected. These are the core features that define the "Kotlin experience."

---

## 5. Interop Model

**Question:** What does "100% bidirectional interop" mean concretely?

| Option                    | Description                                                                      | Chosen? |
|---------------------------|----------------------------------------------------------------------------------|---------|
| Seamless import           | Bak imports Go packages directly. Go imports Bak-generated code. No wrappers.    | No      |
| **Import + type mapping** | Like above, but Bak wraps Go types with null-safety and immutability guarantees. | YES     |
| Gradual migration         | Focus on Bak→Go first. Go→Bak is Phase 2.                                       | No      |

**Decision:** Import + type mapping. Bak adds a safety layer on top of Go types automatically.
This is a stronger value proposition than pure pass-through — Go types gain Bak's safety
guarantees at the boundary.

**Example:**
```bak
// In Bak: use Go packages directly, with safety
import "net/http"
import "github.com/gin-gonic/gin"

fun main() {
    val r = gin.Default()
    r.GET("/") { c ->
        c.JSON(200, mapOf("msg" to "hello"))
    }
    r.Run()
}
```

```go
// In Go: use Bak-generated code as regular Go
import "myproject/bak_module"

func main() {
    result := bak_module.MyFunction()
}
```

---

## 6. Object-Oriented Model

**Question:** How should Bak handle OOP? Go has structs + interfaces but no classes/inheritance.

| Option                        | Description                                                                  | Chosen? |
|-------------------------------|------------------------------------------------------------------------------|---------|
| **Traits + data classes**     | No class inheritance. Traits (interfaces with default methods) + data        | YES     |
|                               | classes + sealed hierarchies. Compiles to Go interfaces + structs.           |         |
| Full class hierarchy          | Traditional OOP with classes, single inheritance. Harder to map to Go.       | No      |
| Go-style composition enhanced | Keep Go's composition (struct embedding) with syntactic sugar.               | No      |

**Decision:** Traits + data classes. Perfect middle ground between Go's minimalism and Kotlin's
expressiveness. Maps cleanly to Go interfaces + structs under the hood.

**Example:**
```bak
trait Drawable {
    fun draw()
    fun description(): String = "drawable"  // default impl
}

data class Circle(val radius: Float) : Drawable {
    override fun draw() { /* ... */ }
}

sealed class Shape {
    data class Rect(val w: Float, val h: Float)
    data class Circle(val r: Float)
}

// exhaustive matching
when (shape) {
    is Shape.Rect  -> computeRectArea(shape.w, shape.h)
    is Shape.Circle -> computeCircleArea(shape.r)
}
```

---

## 7. Error Handling

**Question:** How should Bak handle errors? Go's `if err != nil` is verbose.

| Option                  | Description                                                                  | Chosen? |
|-------------------------|------------------------------------------------------------------------------|---------|
| **Result<T, E> type**   | Like Rust. Errors wrapped in type-safe container. `?` operator for early     | YES     |
|                         | return. Combined with pattern matching, eliminates if-err-nil.               |         |
| Try/catch exceptions    | Traditional Kotlin/Java style. Departure from Go's explicit error philosophy. | No      |
| Both (multi-paradigm)   | Result as default + try/catch sugar. More complex compiler.                  | No      |

**Decision:** Result<T, E> with `?` operator. Stays aligned with Go's "errors are values" philosophy
while eliminating the boilerplate. Pattern matching makes error handling elegant.

**Example:**
```bak
fun readFile(path: String): Result<String, IOError> { /* ... */ }

// Pattern matching
when (val result = readFile("data.txt")) {
    is Ok  -> println(result.value)
    is Err -> log.error(result.error)
}

// ? operator (early return)
fun process(): Result<Data, Error> {
    val content = readFile("data.txt")?
    val parsed = parse(content)?
    return Ok(parsed)
}
```

---

## 8. Additional Features (User Additions)

After the initial feature selection, additional features were identified:

### 8.1 Default Parameter Values
```bak
fun greet(name: String, greeting: String = "Hello"): String = "$greeting, $name"
val msg = greet("Alice")  // uses default: "Hello, Alice"
```

### 8.2 Option Type (Maybe/Some/None) with Destructuring
Unlike Kotlin which lacks proper Option destructuring, Bak has Rust-inspired Option:
```bak
val opt: Option<Int> = Some(42)
when (opt) {
    is Some -> println("Got ${opt.value}")  // destructured
    is None -> println("Nothing")
}
```

### 8.3 Go's defer
Preserved directly from Go:
```bak
defer { file.close() }
```

### 8.4 Go's Channels and Goroutines
Preserved from Go:
```bak
val ch = Channel<String>(10)
go { ch.send("hello") }
val msg = ch.receive()
```

### 8.5 Capitalize Conventions
Follow Go's convention: uppercase = exported (public), lowercase = unexported (private).
This prevents friction when interoperating with Go code.

---

## 9. Completeness Review

After establishing the core features, a comprehensive review identified 12 additional features needed:

| #  | Feature                     | Decision                                                         |
|----|-----------------------------|------------------------------------------------------------------|
| 1  | Generics syntax             | Use `<T>` (Kotlin-style), more familiar than Go's `[T]`         |
| 2  | Smart casts                 | Yes — after `is` check, auto-narrow the type                     |
| 3  | Destructuring declarations  | Yes — `val (name, age) = user`                                   |
| 4  | Ranges                      | Yes — `for i in 0..10`, `0 until n`                              |
| 5  | Enums                       | Yes — `enum class Direction { North, South, East, West }`        |
| 6  | Go error auto-wrapping      | Go `(T, error)` auto-converts to `Result<T, E>` at boundary     |
| 7  | Go multi-return             | Map to `Pair<T, U>` or tuples + destructuring                    |
| 8  | Structural typing           | Yes — preserve Go's strength (no explicit `implements`)          |
| 9  | Type inference               | Full local inference like Kotlin: `val x = 42` inferred as Int  |
| 10 | Operator overloading        | Convention-based like Kotlin: `plus()`, `minus()`, `get()`       |
| 11 | Scope functions             | Library functions (not language primitives): let, apply, also     |
| 12 | Properties (get/set)        | Custom getters/setters, compile to Go methods                    |

---

## 10. Build Approach

**Question:** How should we build the compiler incrementally?

| Option                                | Description                                                        | Chosen? |
|---------------------------------------|--------------------------------------------------------------------|---------|
| **Walking Skeleton (vertical slices)** | Full pipeline from day 1, one feature at a time. Always working.  | YES     |
| Full Grammar First                    | Complete parser first, then backend incrementally.                 | No      |
| REPL-Driven                           | Interactive playground first, full compiler later.                 | No      |

**Decision:** Walking Skeleton. End-to-end vertical slices. XP/TDD-friendly. You always have a
working compiler. Each milestone extends all pipeline phases simultaneously.

---

## 11. Compiler Architecture

```
.bak source → Lexer → Parser → Bak AST → Desugar → Type Check → Go Codegen → .go source → go build
```

### Pipeline phases

| Phase          | Approach                                                              |
|----------------|-----------------------------------------------------------------------|
| Lexer          | Hand-written scanner in Go                                            |
| Parser         | Hand-written recursive descent (like Go's own compiler)               |
| Desugar        | Default params → overloads, ranges → iterators, `${}` → `Sprintf`   |
| Type Check     | Hindley-Milner inference, null safety, exhaustiveness, smart casts    |
| Go Codegen     | Hybrid: `go/ast` for structs/functions, templates for complex patterns|
| Validate       | Run `go/types` on generated output for extra safety                  |

### Key Bak → Go mappings

| Bak construct          | Compiles to Go                                          |
|------------------------|---------------------------------------------------------|
| `data class`           | struct + `Equal()`, `String()`, `Copy()` methods        |
| `trait`                | interface (+ hidden struct for default impls)            |
| `sealed class`         | interface + unexported marker method                    |
| `when (x) { is ... }`  | `switch v := x.(type) { case ... }`                    |
| `Result<T, E>`         | `(T, error)` at Go boundary, wrapped internally         |
| `Option<T>`            | `*T` or `(T, bool)` at Go boundary                     |
| `val`                  | mutation enforced by compiler (Go lacks const vars)      |
| `fun X.ext()`          | `func ext(self X)` free function                        |
| `x?.method()`          | `if x != nil { x.method() }`                           |
| `x ?: default`         | `if x != nil { x } else { default }`                   |
| `?` operator           | `if err != nil { return ..., err }`                     |
| `defer`                | `defer` (pass-through)                                  |
| `go { ... }`           | `go func() { ... }()`                                   |
| `chan<T>`               | `chan T`                                                |
| default params         | multiple function signatures or wrapper func            |

---

## 12. Research: Prior Art

The design was informed by research on existing Go transpilers:

### Borgo (most relevant)
- Active project, Rust-inspired syntax, written in Rust
- Has ADTs, pattern matching, Option/Result, `?` operator
- Transpiles to Go source
- Uses Hindley-Milner type inference
- https://github.com/borgo-lang/borgo

### Oden (best architecture documentation)
- Archived (2016), ML/Haskell-inspired, written in Haskell
- 8+ phase compiler pipeline with parameterized AST
- Excellent reference for compiler architecture
- https://github.com/oden-lang/oden

### Key insights from research
- 8/10 top languages use hand-written recursive descent parsers
- Go itself switched from yacc to hand-written in Go 1.6 (18% speed gain)
- Go's `go/ast` + `go/printer` + `go/types` packages form a complete code generation toolkit
- Kotlin's K2 compiler uses 16 sequential FIR phases — phased resolution with strict ordering invariants

Full research document: `./tmp/bak-research.md`

---

## 13. Walking Skeleton Milestones

| Milestone | Feature                             | Key deliverable                                    |
|-----------|-------------------------------------|----------------------------------------------------|
| M0        | Project Bootstrap                   | `fun main() { println("Hello") }` compiles to Go  |
| M1        | Data Classes                        | struct + generated methods + destructuring          |
| M2        | Functions + Type Inference          | Default params, expression bodies, val/var, `${}`  |
| M3        | Null Safety + Option                | `T?`, `?.`, `!!`, `?:`, `Some`/`None`             |
| M4        | Traits + Sealed + Pattern Matching  | Interfaces with defaults, exhaustive `when`         |
| M5        | Result + Error Handling             | `Result<T,E>`, `?` operator, Go error interop      |
| M6        | Extension Functions + FP            | Extensions, lambdas, collection HOFs               |
| M7        | Go Interop + Concurrency            | Import Go packages, `go{}`, `chan<T>`, `defer`     |
| M8        | Generics + Enums + Ranges + Polish  | `<T>`, enum class, ranges, operator overloading    |

---

## 14. Complete Syntax Reference

```bak
// ============================================
// Bak Language — Complete Syntax Reference
// ============================================

// --- Variables ---
val name: String = "Alice"          // immutable
var count: Int = 0                  // mutable
val inferred = 42                   // type inferred as Int

// --- Null Safety ---
val safe: String = "hello"          // non-null by default
val maybe: String? = null           // nullable
val len = maybe?.length ?: 0        // safe call + elvis
val forced = maybe!!                // assert non-null (throws if null)

// --- Option Type ---
val opt: Option<Int> = Some(42)
val nothing: Option<String> = None

when (opt) {
    is Some -> println("Got ${opt.value}")
    is None -> println("Nothing")
}

// --- Functions ---
fun greet(name: String): String = "Hello, $name"
fun greetFormal(name: String, title: String = "Mr."): String = "$title $name"

// --- Data Classes ---
data class User(val name: String, val age: Int)
val user = User("Alice", 30)
val (name, age) = user              // destructuring
val copy = user.copy(age = 31)

// --- Traits ---
trait Serializable {
    fun serialize(): ByteArray
    fun contentType(): String = "application/json"  // default impl
}

// --- Sealed Classes ---
sealed class AuthResult {
    data class Success(val user: User) : AuthResult()
    data class Failure(val reason: String) : AuthResult()
}

// --- Pattern Matching ---
when (result) {
    is AuthResult.Success -> println("Welcome, ${result.user.name}")
    is AuthResult.Failure -> println("Failed: ${result.reason}")
    // exhaustive — compiler error if a case is missing
}

// --- Smart Casts ---
fun process(x: Any) {
    if (x is String) {
        println(x.length)  // x is automatically narrowed to String
    }
}

// --- Result Type + ? Operator ---
fun loadConfig(path: String): Result<Config, IOError> {
    val data = readFile(path)?       // early return on error
    val config = parse(data)?
    return Ok(config)
}

// --- Extension Functions ---
fun String.IsPalindrome(): Boolean = this == this.reversed()

// --- Lambdas and HOFs ---
val doubled = listOf(1, 2, 3).map { it * 2 }
val evens = numbers.filter { it % 2 == 0 }
val sum = numbers.reduce { acc, n -> acc + n }

// --- String Interpolation ---
println("User: ${user.name}, age: ${user.age}")
println("Simple: $name")

// --- Enums ---
enum class Direction { North, South, East, West }

// --- Ranges ---
for (i in 0..10) { println(i) }
for (i in 0 until n) { println(i) }

// --- Generics ---
fun <T : Comparable<T>> Max(a: T, b: T): T = if (a > b) a else b

// --- Go Concurrency (preserved) ---
val ch = Channel<String>(10)
go { ch.send("hello") }
val msg = ch.receive()

defer { cleanup() }

// --- Go Interop ---
import "net/http"
import "fmt"
// Use Go packages directly — types are auto-wrapped with Bak safety
```
