# Proposal 001: `match` keyword with basic destructuring

**Status:** Proposed
**Date:** 2026-02-28

---

## Summary

Replace `when` (Kotlin-style) with `match` (Rust/Scala-style) for pattern matching.
Add basic destructuring in match arms. Keep `when` only for conditional branching
without a subject.

## Motivation

- `when` conveys "conditional branching" — not pattern deconstruction
- `match` says "decompose this value against patterns" — clearer intent
- Destructuring in match arms eliminates the need to access fields after smart casts

## Proposed Design

### `match` — pattern matching with destructuring

```kotlin
// Destructure sealed class variants
val area = match (shape) {
    is Circle(r) -> 3.14 * r * r
    is Rectangle(w, h) -> w * h
    is Triangle(b, h) -> 0.5 * b * h
}

// Destructure Option
match (opt) {
    is Some(value) -> println(value)
    is None -> println("nothing")
}

// Destructure Result
match (LoadUser(42)) {
    is Ok(user) -> println(user.name)
    is Err(error) -> println(error.message)
}

// Exhaustive — compiler error if a case is missing
match (authResult) {
    is Success(user, token) -> setSession(user, token)
    is Failure(reason) -> showError(reason)
    is TwoFactorRequired(id) -> show2FA(id)
}

// Wildcard
match (pair) {
    is Pair(_, 0) -> println("second is zero")
    is Pair(a, b) -> println("$a, $b")
}
```

### `when` — conditional branching (no subject, no destructuring)

```kotlin
val label = when {
    age < 18  -> "minor"
    age < 65  -> "adult"
    else      -> "senior"
}
```

## Scope (what is NOT included)

These are explicitly deferred to a future proposal if needed:

- Nested destructuring: `is Ok(User(name, age))`
- Guards: `is Circle(r) if r > 100`
- Or-patterns: `is North | is South`
- Literal patterns: `200 -> "OK"`
- Tuple matching: `(0, 0) -> "origin"`
- Binding with `@`: `is Some(x @ Int)`
- Range patterns: `in 500..599`

## Alternatives Considered

| Option                          | Level               | Verdict      |
|---------------------------------|----------------------|--------------|
| Keep `when` (Kotlin-style)      | No destructuring     | Rejected — too verbose, wrong semantics |
| `match` + basic destructuring   | Fields in patterns   | **Chosen**   |
| `match` + nested + guards       | Deep patterns        | Deferred — adds compiler complexity     |
| Full Rust-style `match`         | Everything           | Deferred — overkill for v0.1            |

## Impact on Other Features

- Smart casts still work inside `match` arms (narrowing after `is`)
- `match` is an expression (returns a value)
- Exhaustiveness checking applies to sealed classes and enums
- `when` without subject remains for conditional logic

## Bak → Go Mapping

| Bak                              | Go                                         |
|----------------------------------|--------------------------------------------|
| `match (x) { is Type(a) -> }` | `switch v := x.(type) { case Type: a := v.Field }` |
| `when { cond -> }`              | `switch { case cond: }`                   |
