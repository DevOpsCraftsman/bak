# Bak

**A modern language that compiles to Go.**

Bak is to Go what Kotlin is to Java, and TypeScript is to JavaScript:
a safer, more expressive layer over a proven runtime —
without giving up what makes Go great.

## Why

Go is fast, simple, and pragmatic. But simplicity has costs:

| Pain point                  | What Bak brings                                      |
|-----------------------------|------------------------------------------------------|
| Verbose `if err != nil`     | `Result<T, E>` with `?` operator                    |
| No null safety              | Non-null by default, `T?` opt-in, `Option<T>` for FP |
| Limited type expressiveness | Sealed classes, generics `<T>`, pattern matching     |
| Boilerplate structs         | Data classes with auto `Equal`, `String`, `Copy`     |
| No expression syntax        | Everything is an expression                          |
| Weak FP support             | Lambdas, HOFs, extension functions, trailing lambdas |

## Design goals

### Minimal overhead

Bak compiles to readable Go source code, then uses the standard Go toolchain.
The compilation overhead must stay in the same order of magnitude as `go build`.
No heavy IR, no LLVM — just a fast transpiler that emits `.go` files.

### No innovation

Bak invents nothing. Every feature is a proven idiom borrowed from an existing,
battle-tested language — primarily **Kotlin** and **Rust**, occasionally others.

If it hasn't already worked at scale in a real ecosystem, it doesn't belong in Bak.
The goal is assembly, not invention: pick the best parts, make them fit together
on top of Go, and ship.

### Better ergonomics, without going too far

We borrow selectively — only where a feature genuinely reduces friction.
Bak is not a research language.
If a feature doesn't pull its weight in everyday code, it doesn't get in.

### Fewer errors by construction

- **Non-null by default.** `String` is never nil. `String?` is explicitly nullable.
- **Result and Option types.** No more `(T, error)` returns you forget to check.
- **Exhaustive pattern matching.** The compiler catches missing branches.
- **Immutability preferred.** `val` by default, `var` when you need it.

### Token-optimised for AI-assisted coding

LLMs generate and read Bak. The syntax is designed to be:

- **Low-noise.** Less boilerplate means fewer tokens to generate and parse.
- **Unambiguous.** Clear keywords reduce hallucination risk.
- **Predictable.** Consistent patterns let models infer the right structure.

A `data class` declaration replaces 40+ lines of Go struct boilerplate —
that's real savings when an agent is writing or reviewing code at scale.

### Full Go interop

Like Kotlin on the JVM, Bak gives you the entire Go ecosystem:

- Import any Go package: `import "net/http"`, `import "github.com/gin-gonic/gin"`.
- Generated `.go` files are standard Go — importable from any Go project.
- Go's `(T, error)` returns auto-wrap to `Result<T, E>` at the boundary.
- Adopt Bak incrementally: one file, one module, one team at a time.


## Quick taste

```kotlin
import "net/http"

data class User(val name: String, val age: Int)

sealed class AuthResult {
    data class Success(val user: User) : AuthResult()
    data class Failure(val reason: String) : AuthResult()
}

fun findUser(id: Int): Result<User, Error> {
    val row = db.query(id)?   // early return on error
    return Ok(User(row.name, row.age))
}

fun main() {
    val result = findUser(42)

    when (result) {
        is Ok  -> println("Hello, ${result.value.name}")
        is Err -> println("Error: ${result.error}")
    }

    // null safety
    val name: String? = getName()
    val greeting = name?.uppercase() ?: "Anonymous"

    // FP
    val evens = numbers.filter { it % 2 == 0 }

    // Go concurrency, untouched
    val ch = Channel<String>(10)
    go { ch.send("hello") }
    val msg = ch.receive()
}
```

## Inspirations

| Source  | What we take                                                   |
|---------|----------------------------------------------------------------|
| **Go**  | Goroutines, channels, fast compilation, simple deployment      |
| Kotlin  | Data classes, null safety, `when`, extension functions, syntax |
| Rust    | `Result<T, E>`, `Option<T>`, `?` operator, pattern matching   |
| Swift   | Sealed types (enums with associated data), trailing closures   |

## Project status

Bak is in the **design phase**. The language specification is documented,
syntax is defined across 23 feature areas, and proposals are being evaluated.

Implementation has not started yet. The compiler will be written in Go,
using a hand-written lexer and recursive-descent parser, targeting Go source output.

See `docs/design/` for the full specification and decision records.

## Roadmap

| Milestone | Feature                              |
|-----------|--------------------------------------|
| M0        | Bootstrap: `fun main() { println("Hello") }` compiles to Go |
| M1        | Data classes + destructuring         |
| M2        | Functions, type inference, `val/var` |
| M3        | Null safety + `Option<T>`           |
| M4        | Interfaces, sealed classes, `when`   |
| M5        | `Result<T, E>` + `?` operator       |
| M6        | Extension functions + lambdas        |
| M7        | Go interop + concurrency             |
| M8        | Generics + enums + polish            |

## License

TBD
