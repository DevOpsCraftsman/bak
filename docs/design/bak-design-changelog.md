# Bak Language — Design Changelog

Tracks design revisions to the language specification. Each entry records
what changed, why, and which documents were updated.

The original discussion is preserved as-is in `2026-02-27-bak-language-design-discussion.md`.

---

## Rev 2 — 2026-02-28: Keywords, Nullability Model, Error Types

### Change 1: `trait` → `interface`

**Before:** Bak used the `trait` keyword for interfaces with default methods.
**After:** Bak uses `interface` (same keyword as Go and Kotlin).

**Rationale:**
- Both Go and Kotlin use `interface` — zero learning curve for either audience
- Kotlin interfaces already support default methods, so no semantic mismatch
- Reduces cognitive overhead for Go developers adopting Bak
- Clearer interop story: a Bak `interface` IS a Go `interface`
- Using `trait` would misleadingly signal that Bak interfaces differ from Go interfaces

**Alternatives considered:**

| Option       | Pros                                          | Cons                                            |
|--------------|-----------------------------------------------|------------------------------------------------|
| `trait`      | Distinct identity, signals "this has defaults" | Misleading (compiles to Go interface anyway)    |
| `interface`  | Familiar to Go+Kotlin devs, accurate mapping  | Less "unique" identity for Bak                  |

### Change 2: Dual nullability model (T? + Option<T>)

**Before:** Both `T?` and `Option<T>` existed but their relationship was undefined.
**After:** Both are first-class, with explicit bridge methods. Clear guidance on when to use each.

**Model:**
- `T?` — ergonomic for simple null handling (`?.`, `?:`, `!!`), Go interop
- `Option<T>` — FP composition (`map`, `flatMap`, `filter`), domain modeling
- Bridge: `T?.toOption()` and `Option<T>.toNullable()` for free conversion

**Rationale:**
- `T?` only (Kotlin-style) limits FP composition — must chain with `?.let {}`
- `Option<T>` only (Rust-style) is verbose for simple null checks
- Both with a bridge gives the best of both worlds
- Developers choose the right tool for the situation

**Alternatives considered:**

| Option              | Pros                                | Cons                                         |
|---------------------|-------------------------------------|----------------------------------------------|
| `T?` only           | Simple, familiar to Kotlin devs     | No `map`/`flatMap`/`filter` on nullables     |
| `Option<T>` only    | Consistent, full FP                 | Verbose for `if x != null` patterns          |
| Both with bridge    | Best of both, developer choice      | Two ways to represent absence (explain once) |

**Open question — tension with "one idiomatic way" principle:**
The dual model gives two ways to represent the absence of a value.
This directly contradicts the design goal of having one obvious way to do things.
Additionally, `T?` is significantly more token-efficient than `Option<T>` —
which matters for AI-assisted coding. It may be better to keep only `T?` with
full FP methods (`map`, `flatMap`, `filter`) attached directly to the nullable type,
eliminating the need for `Option<T>` entirely. Needs further evaluation.

### Change 3: Result<T, E> primary, Either<L, R> in stdlib

**Before:** Only `Result<T, E>` existed.
**After:** `Result<T, E>` is the primary error handling type with a full FP API.
`Either<L, R>` is available in the standard library for generic sum types.

**Result API (built-in):**
- `map`, `flatMap`, `mapError`, `recover`, `fold`, `getOrElse`
- `?` operator for early return
- Auto-wraps Go's `(T, error)` at interop boundary

**Either API (stdlib):**
- `fold`, `map`, `mapLeft`, `bimap`, `swap`
- Converts to/from Result: `toResult()`, `toEither()`
- Used for non-error disjunctions (e.g., "this is either an Int or a String")

**Rationale:**
- `Result` maps perfectly to Go's `(T, error)` convention
- `Ok`/`Err` is clearer than `Left`/`Right` for error handling
- `Either` is more general and useful for cases where both sides are valid values
- Having both covers 100% of use cases without forcing FP abstractions on Go devs
- Arrow (Kotlin) validates this approach — it offers both

**Alternatives considered:**

| Option                           | Pros                            | Cons                                        |
|----------------------------------|---------------------------------|---------------------------------------------|
| `Result<T, E>` only             | Simple, clear intent            | No generic sum type for non-error cases     |
| `Either<L, R>` only             | More general, full FP           | `Left`/`Right` less readable than `Ok`/`Err`|
| Result primary + Either stdlib  | Clear defaults, full coverage   | Two related types to learn                  |

### Files updated

| File                                                             | Changes                                      |
|------------------------------------------------------------------|----------------------------------------------|
| `docs/design/bak-syntax-examples.md`                             | trait→interface, Option bridge, Result API, Either section |
| `ai/plans/todo/2026-02-27-bak-language-design-and-bootstrap.md` | trait→interface, updated milestones M3/M4/M5 |
| `docs/design/2026-02-27-bak-language-design-discussion.md`       | NOT modified (preserved as historical record)|

---

## Rev 1 — 2026-02-27: Initial Design

Initial language specification created through collaborative design session.
See `2026-02-27-bak-language-design-discussion.md` for full record.
