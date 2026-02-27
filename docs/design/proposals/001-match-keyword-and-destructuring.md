# Proposal 001: Pattern Matching — Keyword and Destructuring

**Status:** Proposed
**Date:** 2026-02-28

---

## Summary

Two decisions to make:
1. **Keyword**: `when` vs `match` vs both
2. **Destructuring level**: how powerful should pattern matching be?

Current recommendation: `match` with basic destructuring (Option B below),
`when` kept for conditional branching only.

---

## Decision 1: Keyword — `when` vs `match`

### Option W1: `when` (Kotlin convention)

```kotlin
// Pattern matching with subject
val area = when (shape) {
    is Circle -> 3.14 * shape.radius * shape.radius
    is Rectangle -> shape.width * shape.height
}

// Conditional branching (no subject)
val label = when {
    age < 18  -> "minor"
    age < 65  -> "adult"
    else      -> "senior"
}
```

| Pros                                            | Cons                                                   |
|-------------------------------------------------|--------------------------------------------------------|
| Familiar to Kotlin developers                   | Conflates two different concepts (matching vs branching)|
| One keyword for everything                      | "when" semantically means "conditional", not "decompose"|
| Simpler language — fewer keywords to learn      | Less clear when destructuring is added                 |
| Kotlin already uses `when` with destructuring   | Go devs won't associate `when` with pattern matching   |

### Option W2: `match` (Rust/Scala convention)

```kotlin
// Pattern matching with subject
val area = match (shape) {
    is Circle -> 3.14 * shape.radius * shape.radius
    is Rectangle -> shape.width * shape.height
}

// But what about conditional branching? Need a separate construct:
val label = if (age < 18) "minor" else if (age < 65) "adult" else "senior"
// or keep `when` for this:
val label = when {
    age < 18  -> "minor"
    age < 65  -> "adult"
    else      -> "senior"
}
```

| Pros                                                    | Cons                                                 |
|---------------------------------------------------------|------------------------------------------------------|
| Semantically correct: "match against patterns"          | Rust/Scala devs expect full Rust-level power         |
| Clear intent when destructuring is involved             | Two keywords (`match` + `when` or `if`) for branching|
| Distinct from Go's `switch` — signals Bak's identity   | New keyword to learn for Kotlin developers           |
| Used by Rust, Scala, OCaml, Haskell, Elixir, Swift     |                                                      |

### Option W3: Both `match` and `when` (different purposes)  <-- Recommended

```kotlin
// match — for pattern matching with destructuring on a subject
val area = match (shape) {
    is Circle(r) -> 3.14 * r * r
    is Rectangle(w, h) -> w * h
}

// when — for conditional branching WITHOUT a subject
val label = when {
    age < 18  -> "minor"
    age < 65  -> "adult"
    else      -> "senior"
}
```

| Pros                                                    | Cons                                                 |
|---------------------------------------------------------|------------------------------------------------------|
| Each keyword has a clear, distinct purpose              | Two keywords instead of one                          |
| `match` = decompose a value into patterns               | Developers must learn which to use when              |
| `when` = choose based on boolean conditions             | Kotlin devs may be confused why `when` can't match   |
| No semantic ambiguity                                   |                                                      |
| Both are familiar from other languages                  |                                                      |

---

## Decision 2: Destructuring Level

All examples below use `match` as the keyword. The same patterns would work
with `when` if Option W1 is chosen.

### Option A: No destructuring (Kotlin-style — type dispatch only)

```kotlin
// Type checking, then access fields via smart cast
match (shape) {
    is Shape.Circle -> println(shape.radius)
    is Shape.Rectangle -> println(shape.width * shape.height)
}

// Option — access .value after smart cast
match (opt) {
    is Some -> println(opt.value)
    is None -> println("nothing")
}

// Result — same pattern
match (LoadUser(42)) {
    is Ok  -> println(result.value.name)
    is Err -> println(result.error.message)
}

// Sealed class
match (authResult) {
    is AuthResult.Success -> {
        setSession(authResult.user, authResult.token)
    }
    is AuthResult.Failure -> {
        showError(authResult.reason)
    }
    is AuthResult.TwoFactorRequired -> {
        show2FA(authResult.challengeId)
    }
}

// Enum
match (direction) {
    Direction.North -> "up"
    Direction.South -> "down"
    Direction.East  -> "right"
    Direction.West  -> "left"
}
```

| Pros                                            | Cons                                                          |
|-------------------------------------------------|---------------------------------------------------------------|
| Simplest compiler implementation                | Verbose — must repeat the subject name to access fields       |
| No new syntax to learn beyond `is`              | Doesn't feel like real pattern matching                       |
| Smart casts handle the type narrowing           | Nested access is painful: `result.value.address.city`         |
| Kotlin developers are already used to this      | Misses the main point of using `match` over `when`            |

### Option B: Basic destructuring  <-- Recommended

```kotlin
// Destructure fields directly in the pattern
match (shape) {
    is Circle(radius) -> println(radius)
    is Rectangle(w, h) -> println(w * h)
    is Triangle(base, height) -> 0.5 * base * height
}

// Option destructuring
match (opt) {
    is Some(value) -> println(value)
    is None -> println("nothing")
}

// Result destructuring
match (LoadUser(42)) {
    is Ok(user) -> println(user.name)
    is Err(error) -> println(error.message)
}

// Sealed class — fields extracted directly
match (authResult) {
    is Success(user, token) -> setSession(user, token)
    is Failure(reason) -> showError(reason)
    is TwoFactorRequired(id) -> show2FA(id)
}

// Wildcard — ignore fields you don't need
match (pair) {
    is Pair(_, 0) -> println("second is zero")
    is Pair(a, b) -> println("$a, $b")
}

// Match as expression — returns a value
val area = match (shape) {
    is Circle(r) -> 3.14 * r * r
    is Rectangle(w, h) -> w * h
    is Triangle(b, h) -> 0.5 * b * h
}

// Exhaustive — compiler error if a case is missing
// (works on sealed classes and enums)

// Enum matching (no destructuring needed)
match (direction) {
    Direction.North -> "up"
    Direction.South -> "down"
    Direction.East  -> "right"
    Direction.West  -> "left"
}
```

| Pros                                                    | Cons                                                      |
|---------------------------------------------------------|-----------------------------------------------------------|
| Concise — fields bound directly in the pattern          | Only one level deep (can't destructure nested types)      |
| Covers 90% of real-world use cases                      | No guards (can't add conditions like `if r > 100`)       |
| Reasonable compiler complexity                          | No or-patterns (can't merge similar cases)                |
| Clean Go mapping (type switch + field extraction)       | No literal matching (must use `when` for `200 -> "OK"`)   |
| Familiar to Scala/Kotlin/Swift developers               |                                                           |

### Option C: Nested destructuring + guards

```kotlin
// Everything from Option B, plus:

// Nested destructuring — reach into nested data classes
match (result) {
    is Ok(User(name, age)) -> println("$name is $age")
    is Err(DBError.NotFound(id)) -> println("No user $id")
    is Err(DBError.Timeout(_)) -> retry()
    is Err(e) -> log.error(e)
}

// Guards — additional conditions on patterns
match (shape) {
    is Circle(r) if r > 100.0 -> println("large circle")
    is Circle(r) -> println("small circle: $r")
    is Rectangle(w, h) if w == h -> println("square: $w")
    is Rectangle(w, h) -> println("rect: ${w}x${h}")
}

// Guards on Option
match (opt) {
    is Some(x) if x > 0 -> println("positive: $x")
    is Some(x) -> println("non-positive: $x")
    is None -> println("absent")
}

// Guards on Result
match (LoadUser(42)) {
    is Ok(user) if user.isAdmin -> grantAdmin(user)
    is Ok(user) -> grantBasic(user)
    is Err(e) -> handleError(e)
}

// Wildcard at any depth
match (result) {
    is Ok(User(_, age)) if age >= 18 -> println("adult")
    is Ok(User(name, _)) -> println("minor: $name")
    is Err(_) -> println("error")
}

// Nested sealed class destructuring
sealed class Expr {
    data class Num(val value: Int) : Expr()
    data class Add(val left: Expr, val right: Expr) : Expr()
    data class Mul(val left: Expr, val right: Expr) : Expr()
}

match (expr) {
    is Num(n) -> n
    is Add(Num(a), Num(b)) -> a + b          // nested!
    is Mul(left, Num(0)) -> 0                 // optimization
    is Mul(left, Num(1)) -> eval(left)        // optimization
    is Add(left, right) -> eval(left) + eval(right)
    is Mul(left, right) -> eval(left) * eval(right)
}
```

| Pros                                                    | Cons                                                       |
|---------------------------------------------------------|------------------------------------------------------------|
| Very expressive — handles complex data decomposition    | Significantly more complex compiler (recursive patterns)   |
| Guards eliminate need for nested if/else inside arms    | Exhaustiveness checking with guards is hard                |
| Excellent for recursive data structures (ASTs, trees)   | Nested patterns can become hard to read                    |
| Covers 99% of use cases                                | Go code generation becomes more complex                    |
| Familiar to Rust/Scala/Haskell developers              | Ordering of arms matters more (first match wins with guards)|

### Option D: Full Rust-style match

```kotlin
// Everything from Option C, plus:

// Literal patterns — match on values directly
match (statusCode) {
    200 -> println("OK")
    201 -> println("Created")
    404 -> println("Not found")
    else -> println("Other: $statusCode")
}

// Range patterns
match (score) {
    in 90..100 -> "A"
    in 80..89  -> "B"
    in 70..79  -> "C"
    in 0..69   -> "F"
    else       -> "invalid"
}

// Tuple matching — match on multiple values at once
match (x, y) {
    (0, 0) -> println("origin")
    (0, _) -> println("on y-axis")
    (_, 0) -> println("on x-axis")
    (a, b) if a == b -> println("diagonal")
    (a, b) -> println("point ($a, $b)")
}

// Or-patterns — merge similar cases
match (direction) {
    is North | is South -> println("vertical")
    is East | is West -> println("horizontal")
}

match (event) {
    is KeyDown("Enter") | is KeyDown("Return") -> submit()
    is KeyDown("Escape") | is KeyDown("Esc") -> cancel()
    is KeyDown(key) -> handleKey(key)
    is MouseClick(x, y) -> handleClick(x, y)
}

// Binding with @ — name a sub-pattern
match (opt) {
    is Some(x @ Int) if x > 0 -> println("positive int: $x")
    is Some(s @ String) -> println("string: $s")
    is None -> println("nothing")
}

// Binding a whole variant while also destructuring
match (result) {
    is ok @ Ok(user) if user.isAdmin -> auditLog(ok); grantAdmin(user)
    is Ok(user) -> grantBasic(user)
    is Err(e) -> handleError(e)
}

// Nested literal matching
match (command) {
    is Move(0, 0) -> println("no-op")
    is Move(dx, 0) -> println("horizontal: $dx")
    is Move(0, dy) -> println("vertical: $dy")
    is Move(dx, dy) -> println("diagonal: $dx, $dy")
}

// String pattern matching
match (input) {
    "" -> println("empty")
    "quit" | "exit" | "q" -> shutdown()
    s if s.startsWith("/") -> handleCommand(s)
    s -> echo(s)
}
```

| Pros                                                        | Cons                                                            |
|-------------------------------------------------------------|-----------------------------------------------------------------|
| Maximum expressiveness — handles any decomposition          | Major compiler complexity (literal matching, tuples, @, or)     |
| Literal + range patterns replace `when` entirely            | Go has no equivalent — code generation becomes very complex     |
| Or-patterns reduce code duplication significantly           | Too many features to learn at once                              |
| Tuple matching is very powerful for multi-value logic        | Risk of over-engineering — YAGNI for v0.1                       |
| @ bindings enable naming sub-patterns while destructuring   | Exhaustiveness checking with all features is a research problem |
| Rust/Haskell/OCaml developers feel at home                  | String patterns blur the line between matching and parsing      |

---

## Comparison Matrix

| Feature                  | A (no destruct) | B (basic)    | C (nested+guards) | D (full Rust)   |
|--------------------------|:---------------:|:------------:|:------------------:|:---------------:|
| Type dispatch            | yes             | yes          | yes                | yes             |
| Field destructuring      | no              | yes          | yes                | yes             |
| Wildcards (`_`)          | no              | yes          | yes                | yes             |
| Match as expression      | yes             | yes          | yes                | yes             |
| Exhaustiveness checking  | yes             | yes          | yes (harder)       | yes (very hard) |
| Nested destructuring     | no              | no           | yes                | yes             |
| Guards (`if`)            | no              | no           | yes                | yes             |
| Or-patterns (`\|`)       | no              | no           | no                 | yes             |
| Literal patterns         | no              | no           | no                 | yes             |
| Range patterns           | no              | no           | no                 | yes             |
| Tuple matching           | no              | no           | no                 | yes             |
| @ bindings               | no              | no           | no                 | yes             |
| Compiler complexity      | low             | moderate     | high               | very high       |
| Go codegen complexity    | trivial         | moderate     | complex            | very complex    |
| Covers % of use cases    | ~70%            | ~90%         | ~99%               | 100%            |

---

## Go Code Generation

How each option maps to Go:

### Option A → Go

```kotlin
// Bak
match (shape) {
    is Circle -> println(shape.radius)
    is Rectangle -> println(shape.width)
}
```

```go
// Generated Go
switch v := shape.(type) {
case Circle:
    fmt.Println(v.Radius)
case Rectangle:
    fmt.Println(v.Width)
}
```

### Option B → Go

```kotlin
// Bak
match (shape) {
    is Circle(r) -> println(r)
    is Rectangle(w, h) -> println(w * h)
}
```

```go
// Generated Go
switch v := shape.(type) {
case Circle:
    r := v.Radius
    fmt.Println(r)
case Rectangle:
    w := v.Width
    h := v.Height
    fmt.Println(w * h)
}
```

### Option C → Go

```kotlin
// Bak
match (result) {
    is Ok(User(name, age)) -> println("$name is $age")
    is Err(e) if e.isRetryable() -> retry()
    is Err(e) -> fail(e)
}
```

```go
// Generated Go
switch v := result.(type) {
case Ok:
    user := v.Value
    name := user.Name
    age := user.Age
    fmt.Printf("%s is %d\n", name, age)
case Err:
    e := v.Error
    if e.IsRetryable() {
        retry()
    } else {
        fail(e)
    }
}
```

### Option D → Go

```kotlin
// Bak
match (statusCode) {
    200 -> "OK"
    in 400..499 -> "Client error"
    else -> "Other"
}
```

```go
// Generated Go
switch {
case statusCode == 200:
    return "OK"
case statusCode >= 400 && statusCode <= 499:
    return "Client error"
default:
    return "Other"
}
```

---

## Current Recommendation

**Keyword:** Option W3 — both `match` and `when` (different purposes)
**Destructuring:** Option B — basic destructuring

This gives Bak a clear identity (`match` is distinctly Bak, not Kotlin, not Go) while
keeping compiler complexity manageable. Options C and D can be added incrementally
in future versions without breaking existing code.

### Upgrade path

```
v0.1: Option B (basic destructuring)
v0.2: + guards (if)                    → partial Option C
v0.3: + nested destructuring           → full Option C
v1.0: + or-patterns, literals, ranges  → partial Option D (if demand exists)
```
