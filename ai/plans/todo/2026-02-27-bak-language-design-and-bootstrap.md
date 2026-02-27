# Plan: Bak Language — Kotlin for Go

## Context

Go is a powerful, fast, and simple language — but its verbosity, lack of null safety,
and minimal type system expressiveness leave a gap. Bak fills that gap the way
Kotlin did for Java: more elegant syntax, null safety, richer type system, and
purer OOP/FP constructs — while maintaining **100% bidirectional interop** with Go.

Bak transpiles to Go source code, then uses `go build` for native binaries.
The compiler is written in Go.

## Language Design Summary

### Core identity

Bak = Kotlin's elegance + Go's simplicity + Rust's safety ideas, transpiled to Go.

### Feature set (v0.1)

| Category             | Features                                                                |
|----------------------|-------------------------------------------------------------------------|
| **Safety**           | Non-null by default, `T?` nullable, `?.`, `!!`, `?:` (elvis)           |
| **Option type**      | `Option<T>` = `Some(value)` / `None`, destructuring, pattern matching   |
| **Result type**      | `Result<T, E>` = `Ok(value)` / `Err(error)`, `?` early-return operator  |
| **Data classes**     | Auto `Equal()`, `String()`, `Copy()`, destructuring                     |
| **Traits**           | Interfaces with default method implementations                         |
| **Sealed classes**   | Closed hierarchies, exhaustive `when` matching                          |
| **Pattern matching** | `when (x) { is Type -> ... }` with smart casts                         |
| **Functions**        | Default parameters, expression body, extension functions                |
| **FP**               | Lambdas, trailing lambda syntax, `map`/`filter`/`reduce`, HOFs         |
| **Immutability**     | `val` (immutable) / `var` (mutable)                                     |
| **Type inference**   | Full local inference (`val x = 42`)                                     |
| **Generics**         | `<T>` syntax (Kotlin-style, not Go's `[T]`)                            |
| **Enums**            | `enum class Color { Red, Green, Blue }`                                 |
| **Ranges**           | `for i in 0..10`, `0 until n`                                          |
| **String interp**    | `"Hello, ${name}"` → `fmt.Sprintf`                                     |
| **Properties**       | Custom getters/setters, compile to Go methods                           |
| **Operator overload**| Convention-based: `plus()`, `minus()`, `get()`, `equals()`             |
| **Destructuring**    | `val (name, age) = user`                                                |
| **Smart casts**      | After `is` check, type is narrowed automatically                        |
| **Go concurrency**   | `go { ... }`, `chan<T>`, `select`, `defer` — preserved from Go          |
| **Go conventions**   | Uppercase = exported (public), lowercase = unexported (private)         |
| **Go interop**       | Import Go packages directly; Go imports Bak-generated code as Go        |
| **Error interop**    | Go `(T, error)` auto-wraps to `Result<T, E>` at boundary               |

### Syntax example

```bak
import "net/http"

data class User(val name: String, val age: Int)

trait Greetable {
    fun greet(): String
    fun Formal(): String = "Dear ${greet()}"  // uppercase = exported
}

sealed class AuthResult {
    data class Success(val user: User) : AuthResult()
    data class Failure(val reason: String) : AuthResult()
}

fun FindUser(id: Int, db: Database = defaultDB): Result<User, DBError> {
    val row = db.query(id)?
    return Ok(User(row.name, row.age))
}

fun main() {
    when (val result = FindUser(42)) {
        is Ok -> {
            val (name, age) = result.value
            println("Found: $name, age $age")
        }
        is Err -> println("Error: ${result.error}")
    }

    val name: String? = GetName()
    val greeting = name?.uppercase() ?: "Anonymous"

    val opt: Option<Int> = Some(10)
    when (opt) {
        is Some -> println("Got ${opt.value}")
        is None -> println("Nothing")
    }

    val ch = Channel<String>(10)
    go { ch.send("hello") }
    val msg = ch.receive()
    defer { cleanup() }
}
```

## Compiler Architecture

```
.bak source → Lexer → Parser → Bak AST → Desugar → Type Check → Go Codegen → .go source → go build
```

### Pipeline phases

| Phase          | Responsibility                                                        |
|----------------|-----------------------------------------------------------------------|
| **Lexer**      | Tokenize `.bak` source (hand-written, Go)                             |
| **Parser**     | Recursive descent → Bak AST (hand-written, like Go's own compiler)    |
| **Desugar**    | Default params → overloads, ranges → iterators, `${}` → `Sprintf`    |
| **Type Check** | Hindley-Milner inference, null safety, exhaustiveness, smart casts     |
| **Go Codegen** | Hybrid: `go/ast` for structs/functions, templates for complex patterns|
| **Validate**   | Run `go/types` on generated output for extra safety                   |

### Key Bak → Go mappings

| Bak construct         | Compiles to Go                                          |
|-----------------------|---------------------------------------------------------|
| `data class`          | struct + `Equal()`, `String()`, `Copy()` methods        |
| `trait`               | interface (+ hidden struct for default impls)            |
| `sealed class`        | interface + unexported marker method                    |
| `when (x) { is ... }` | `switch v := x.(type) { case ... }`                    |
| `Result<T, E>`        | `(T, error)` at Go boundary, wrapped internally         |
| `Option<T>`           | `*T` or `(T, bool)` at Go boundary                     |
| `val`                 | mutation enforced by compiler (Go lacks const vars)      |
| `fun X.ext()`         | `func ext(self X)` free function                        |
| `x?.method()`         | `if x != nil { x.method() }`                           |
| `x ?: default`        | `if x != nil { x } else { default }`                   |
| `?` operator          | `if err != nil { return ..., err }`                     |
| `defer`               | `defer` (pass-through)                                  |
| `go { ... }`          | `go func() { ... }()`                                   |
| `chan<T>`              | `chan T`                                                |
| default params        | multiple function signatures or wrapper func            |

## Project structure

```
bak/
├── cmd/bak/           # CLI entry point (bak build, bak run)
├── lexer/             # Tokenizer
├── token/             # Token type definitions
├── parser/            # Recursive descent parser → Bak AST
├── ast/               # Bak AST node definitions
├── desugar/           # Syntax sugar normalization
├── checker/           # Type inference, null checks, exhaustiveness
├── codegen/           # Go source code generation
├── types/             # Bak type system definitions
├── runtime/           # Minimal runtime Go package (Option, Result)
├── stdlib/            # Bak standard library extensions
├── testdata/          # .bak inputs + expected .go golden files
└── go.mod
```

## Walking Skeleton Milestones

Each milestone = working compiler + tests. Pipeline always green.

### Milestone 0: Project Bootstrap
- Go module init, CLI skeleton (`bak build`, `bak run`)
- Lexer + parser + codegen wired end-to-end (empty pipeline)
- Golden file test harness: `.bak` → compare expected `.go`
- **First compilable program:** `fun main() { println("Hello from Bak") }`

### Milestone 1: Data Classes
- `data class User(val name: String, val age: Int)`
- → Go struct + `Equal()`, `String()`, `Copy()` generated methods
- Destructuring: `val (name, age) = user`

### Milestone 2: Functions + Type Inference
- Named functions, expression bodies, default parameters
- String interpolation `"$name"` and `"${expr}"`
- Local type inference: `val x = 42` inferred as `Int`
- `val` / `var` distinction (compiler-enforced immutability)

### Milestone 3: Null Safety + Option
- `T?` nullable types, `?.`, `!!`, `?:` operators
- `Option<T>` = `Some(value)` / `None`
- Pattern matching on Option with destructuring

### Milestone 4: Traits + Sealed Classes + Pattern Matching
- `trait` with default impls → Go interfaces
- `sealed class` → closed type hierarchy
- `when` expression with exhaustive `is` checks
- Smart casts after `is` narrowing

### Milestone 5: Result + Error Handling
- `Result<T, E>` = `Ok` / `Err`
- `?` operator for early return on error
- Go interop: auto-wrap `(T, error)` → `Result<T, E>`

### Milestone 6: Extension Functions + FP
- Extension functions on any type (including Go stdlib)
- Lambdas, trailing lambda syntax
- Collection HOFs: `map`, `filter`, `reduce`, `forEach`

### Milestone 7: Go Interop + Concurrency
- Import Go packages directly
- Go imports Bak-generated packages
- `go { }`, `chan<T>`, `select`, `defer`
- Type mapping layer (Go types → Bak's safety guarantees)

### Milestone 8: Generics + Enums + Ranges + Polish
- `<T>` generics syntax
- `enum class`
- `for i in 0..n` ranges
- Operator overloading (convention-based)
- Custom properties (get/set)
- Scope functions as stdlib

## Implementation approach: Milestone 0

**This is what we'll build first.** A walking skeleton.

### Step 1: Initialize Go module and project structure
- `go mod init github.com/alexandrepoitevin/bak`
- Create directory structure: `cmd/bak/`, `token/`, `lexer/`, `parser/`, `ast/`, `codegen/`, `testdata/`
- Create `runtime/` Go package with placeholder `Option` and `Result` types

### Step 2: Define token types (`token/`)
- Token type enum: keywords (`fun`, `val`, `var`, `data`, `class`, etc.)
- Operators: `+`, `-`, `*`, `/`, `=`, `==`, `!=`, `?.`, `!!`, `?:`, `?`
- Delimiters: `(`, `)`, `{`, `}`, `,`, `:`, `.`
- Literals: String, Int, Float, Bool
- Identifiers

### Step 3: Implement lexer (`lexer/`)
- Hand-written scanner
- TDD: write test cases first, then implement
- Handle string interpolation tokenization (deferred — just string literals for M0)
- Position tracking (file, line, column) for error messages

### Step 4: Define Bak AST nodes (`ast/`)
- For M0, only: `File`, `FunDecl`, `CallExpr`, `StringLiteral`, `Identifier`
- Each node carries source position for error reporting

### Step 5: Implement parser (`parser/`)
- Hand-written recursive descent
- TDD: parse `.bak` source → assert AST structure
- For M0: parse `fun main() { println("...") }` only

### Step 6: Implement Go code generator (`codegen/`)
- Takes Bak AST, emits Go source string
- For M0: `fun main() { println("...") }` → `package main\nimport "fmt"\nfunc main() { fmt.Println("...") }`
- Run `format.Source()` on output for clean formatting
- Validate with `go/types` (optional for M0, but wire the hook)

### Step 7: Wire CLI (`cmd/bak/`)
- `bak build <file.bak>` → generates `.go` files → runs `go build`
- `bak run <file.bak>` → generates + builds + runs

### Step 8: Golden file test harness
- `testdata/hello.bak` + `testdata/hello.go.expected`
- Test runner: compile `.bak` → compare output with `.go.expected`
- First passing test: hello world

## BDD step

Before implementing each milestone, run a quick BDD session:
- 1 happy path scenario
- 2-3 edge cases
- Write failing acceptance test first, then implement

Example for Milestone 0:
```gherkin
Scenario: Compile hello world
  Given a file "hello.bak" containing "fun main() { println("Hello from Bak") }"
  When I run "bak build hello.bak"
  Then a file "hello.go" is generated
  And it contains valid Go code
  And running "go run hello.go" outputs "Hello from Bak"
```

## Verification

- Each milestone: golden file tests pass (`bak build` produces expected Go)
- Each milestone: generated Go compiles with `go build`
- Each milestone: generated Go runs and produces expected output
- CI: `go test ./...` on every commit

## Reference materials

- **Borgo** (closest prior art): https://github.com/borgo-lang/borgo
- **Oden** (best architecture docs): https://github.com/oden-lang/oden
- **Research doc**: `./tmp/bak-research.md`
- **Go AST tooling**: https://pkg.go.dev/go/ast
- **Go type checker**: https://pkg.go.dev/go/types
