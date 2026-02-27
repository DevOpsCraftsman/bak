# Bak Language — Syntax Examples

Complete reference of all planned language features with examples.

---

## 1. Variables and Type Inference

```kotlin
// Immutable (preferred)
val name: String = "Alice"
val age: Int = 30
val pi: Float = 3.14

// Type inference — compiler deduces the type
val inferred = 42              // Int
val greeting = "hello"         // String
val flag = true                // Boolean

// Mutable
var counter: Int = 0
counter = counter + 1
```

---

## 2. Null Safety

```kotlin
// Non-null by default — this CANNOT be null
val name: String = "Alice"

// Nullable — must use T? explicitly
val maybeName: String? = null
val anotherName: String? = "Bob"

// Safe call operator — returns null if receiver is null
val len: Int? = maybeName?.length

// Elvis operator — provide default when null
val displayName = maybeName ?: "Anonymous"

// Chained safe calls
val city: String? = user?.address?.city

// Non-null assertion — throws if null (use sparingly!)
val forced: String = maybeName!!

// Safe call + elvis combined
val upperName = maybeName?.uppercase() ?: "UNKNOWN"
```

---

## 3. Option Type (Some / None)

```kotlin
// Explicit Option type — like Rust's Option<T>
val present: Option<Int> = Some(42)
val absent: Option<Int> = None

// Pattern matching with destructuring
when (present) {
    is Some -> println("Got: ${present.value}")
    is None -> println("Nothing here")
}

// Mapping over Option
val doubled = present.map { it * 2 }        // Some(84)
val nothing = absent.map { it * 2 }          // None

// Unwrap with default
val value = present.unwrapOr(0)              // 42
val fallback = absent.unwrapOr(0)            // 0

// Option from nullable
val opt: Option<String> = Option.from(maybeName)

// Nested destructuring
data class Config(val timeout: Option<Int>, val retries: Option<Int>)

fun ApplyConfig(config: Config) {
    when (val t = config.timeout) {
        is Some -> setTimeout(t.value)
        is None -> setTimeout(30)  // default
    }
}
```

---

## 4. Result Type and Error Handling

```kotlin
// Result<T, E> — Ok or Err
fun ReadFile(path: String): Result<String, IOError> {
    // ...
}

// Pattern matching on Result
when (val result = ReadFile("config.yaml")) {
    is Ok  -> println("Content: ${result.value}")
    is Err -> println("Failed: ${result.error}")
}

// ? operator — early return on error
fun LoadConfig(path: String): Result<Config, AppError> {
    val content = ReadFile(path)?               // returns Err early if fails
    val parsed = ParseYaml(content)?            // same here
    val validated = Validate(parsed)?           // and here
    return Ok(validated)
}

// Chaining Results
fun ProcessData(input: String): Result<Output, Error> {
    return Parse(input)
        .map { Transform(it) }
        .flatMap { Validate(it) }
}

// Go interop — Go's (T, error) auto-wraps to Result<T, E>
import "os"

fun ReadGoFile(path: String): Result<ByteArray, Error> {
    return os.ReadFile(path)  // Go returns ([]byte, error) → auto-wrapped
}
```

---

## 5. Functions

```kotlin
// Standard function
fun Add(a: Int, b: Int): Int {
    return a + b
}

// Expression body (single expression, no braces needed)
fun Add(a: Int, b: Int): Int = a + b

// Return type inference
fun Multiply(a: Int, b: Int) = a * b

// Default parameter values
fun Greet(name: String, greeting: String = "Hello"): String {
    return "$greeting, $name!"
}

// Calling with defaults
val msg1 = Greet("Alice")                // "Hello, Alice!"
val msg2 = Greet("Bob", "Hi")            // "Hi, Bob!"

// Named arguments
val msg3 = Greet(greeting = "Hey", name = "Charlie")

// Unit return (no meaningful return value)
fun Log(message: String) {
    println(message)
}
```

---

## 6. String Interpolation

```kotlin
val name = "Alice"
val age = 30

// Simple variable interpolation
println("Hello, $name")

// Expression interpolation
println("In 5 years: ${age + 5}")
println("Uppercase: ${name.uppercase()}")

// Multi-line strings
val json = """
    {
        "name": "$name",
        "age": $age
    }
"""
```

---

## 7. Data Classes

```kotlin
// Auto-generates: Equal(), String(), Copy(), hash
data class User(val name: String, val age: Int)

// Creating instances
val alice = User("Alice", 30)
val bob = User("Bob", 25)

// Auto-generated String()
println(alice)  // User(name=Alice, age=30)

// Auto-generated Equal()
println(alice == User("Alice", 30))  // true

// Copy with modifications
val olderAlice = alice.copy(age = 31)

// Destructuring
val (name, age) = alice
println("$name is $age years old")

// Nested data classes
data class Address(val street: String, val city: String, val zip: String)
data class Person(val name: String, val address: Address)

val person = Person("Alice", Address("123 Main St", "Springfield", "62701"))
val (personName, address) = person
val (street, city, zip) = address
```

---

## 8. Traits

```kotlin
// Basic trait (compiles to Go interface)
trait Drawable {
    fun Draw()
    fun BoundingBox(): Rect
}

// Trait with default implementation
trait Serializable {
    fun Serialize(): ByteArray
    fun ContentType(): String = "application/json"  // default impl
    fun SerializeToString(): String = String(Serialize())  // uses other method
}

// Implementing a trait
data class Circle(val x: Float, val y: Float, val radius: Float) : Drawable {
    override fun Draw() {
        // drawing logic
    }
    override fun BoundingBox(): Rect {
        return Rect(x - radius, y - radius, radius * 2, radius * 2)
    }
}

// Multiple traits
data class Widget(val id: String) : Drawable, Serializable {
    override fun Draw() { /* ... */ }
    override fun BoundingBox() = Rect(0, 0, 100, 50)
    override fun Serialize() = id.toByteArray()
    // ContentType() uses default: "application/json"
}

// Trait as parameter type
fun Render(item: Drawable) {
    item.Draw()
}

// Trait bounds on generics
fun <T : Serializable> Save(item: T) {
    val bytes = item.Serialize()
    WriteToFile(bytes)
}
```

---

## 9. Sealed Classes and Pattern Matching

```kotlin
// Sealed class — closed hierarchy, compiler knows ALL variants
sealed class Shape {
    data class Circle(val radius: Float) : Shape()
    data class Rectangle(val width: Float, val height: Float) : Shape()
    data class Triangle(val base: Float, val height: Float) : Shape()
}

// Exhaustive pattern matching — compiler error if a case is missing
fun Area(shape: Shape): Float = when (shape) {
    is Shape.Circle    -> 3.14159 * shape.radius * shape.radius
    is Shape.Rectangle -> shape.width * shape.height
    is Shape.Triangle  -> 0.5 * shape.base * shape.height
}

// Sealed for domain modeling
sealed class AuthResult {
    data class Success(val user: User, val token: String) : AuthResult()
    data class Failure(val reason: String) : AuthResult()
    data class TwoFactorRequired(val challengeId: String) : AuthResult()
}

fun HandleAuth(result: AuthResult) {
    when (result) {
        is AuthResult.Success -> {
            SetSession(result.user, result.token)
            Redirect("/dashboard")
        }
        is AuthResult.Failure -> {
            ShowError(result.reason)
        }
        is AuthResult.TwoFactorRequired -> {
            Show2FAPrompt(result.challengeId)
        }
    }
}

// Nested when with guards
fun Classify(value: Int): String = when {
    value < 0   -> "negative"
    value == 0  -> "zero"
    value < 100 -> "small"
    else        -> "large"
}
```

---

## 10. Smart Casts

```kotlin
// After an `is` check, the compiler narrows the type automatically
fun Describe(x: Any): String {
    if (x is String) {
        // x is automatically treated as String here
        return "String of length ${x.length}"
    }
    if (x is Int) {
        // x is automatically treated as Int here
        return "Integer: ${x * 2}"
    }
    return "Unknown"
}

// Smart casts in when expressions
fun Process(input: Any) {
    when (input) {
        is String -> println(input.uppercase())     // String methods available
        is Int    -> println(input + 1)              // Int operations available
        is List   -> println(input.size)             // List methods available
    }
}

// Smart casts with null checks
fun SafeProcess(value: String?) {
    if (value != null) {
        // value is narrowed to non-null String
        println(value.length)
    }
}
```

---

## 11. Extension Functions

```kotlin
// Add methods to any type without modifying it
fun String.IsPalindrome(): Boolean {
    val cleaned = this.lowercase().filter { it.isLetter() }
    return cleaned == cleaned.reversed()
}

// Usage
val result = "racecar".IsPalindrome()  // true

// Extension on Int
fun Int.IsEven(): Boolean = this % 2 == 0

val check = 42.IsEven()  // true

// Extension on Go stdlib types
fun http.Request.BearerToken(): String? {
    val header = this.Header.Get("Authorization")
    return if (header.startsWith("Bearer ")) header.substring(7) else null
}

// Extension with generics
fun <T> List<T>.SecondOrNull(): T? {
    return if (this.size >= 2) this[1] else null
}
```

---

## 12. Lambdas and Higher-Order Functions

```kotlin
// Lambda syntax
val double = { x: Int -> x * 2 }
val sum = { a: Int, b: Int -> a + b }

// Higher-order function (function that takes a function)
fun Apply(value: Int, transform: (Int) -> Int): Int {
    return transform(value)
}
val result = Apply(5, double)  // 10

// Trailing lambda syntax
val numbers = listOf(1, 2, 3, 4, 5)

val doubled = numbers.map { it * 2 }             // [2, 4, 6, 8, 10]
val evens = numbers.filter { it % 2 == 0 }       // [2, 4]
val sum = numbers.reduce { acc, n -> acc + n }    // 15
val names = users.map { it.name }                 // project a field

// Chaining
val result = numbers
    .filter { it > 2 }
    .map { it * 10 }
    .reduce { acc, n -> acc + n }  // 120

// forEach
numbers.forEach { println(it) }

// `it` — implicit single parameter name
val lengths = words.map { it.length }

// Multi-line lambda
val processor = { input: String ->
    val trimmed = input.trim()
    val upper = trimmed.uppercase()
    upper  // last expression is the return value
}

// Function references
fun IsPositive(n: Int): Boolean = n > 0
val positives = numbers.filter(::IsPositive)
```

---

## 13. Enums

```kotlin
// Simple enum
enum class Direction {
    North, South, East, West
}

// Enum with properties
enum class Color(val hex: String) {
    Red("#FF0000"),
    Green("#00FF00"),
    Blue("#0000FF")
}

// Using enums
val dir = Direction.North
val color = Color.Red
println(color.hex)  // #FF0000

// Exhaustive matching on enums
fun Arrow(dir: Direction): String = when (dir) {
    Direction.North -> "↑"
    Direction.South -> "↓"
    Direction.East  -> "→"
    Direction.West  -> "←"
}
```

---

## 14. Generics

```kotlin
// Generic function
fun <T> First(list: List<T>): Option<T> {
    return if (list.isEmpty()) None else Some(list[0])
}

// Generic with constraint
fun <T : Comparable<T>> Max(a: T, b: T): T = if (a > b) a else b

// Generic data class
data class Pair<A, B>(val first: A, val second: B)

val pair = Pair("age", 30)
val (key, value) = pair  // destructuring

// Generic trait
trait Repository<T> {
    fun FindById(id: String): Result<T, Error>
    fun Save(entity: T): Result<Unit, Error>
    fun Delete(id: String): Result<Unit, Error>
}

// Implementing generic trait
data class UserRepo(val db: Database) : Repository<User> {
    override fun FindById(id: String): Result<User, Error> {
        return db.Query("SELECT * FROM users WHERE id = ?", id)?
    }
    override fun Save(entity: User): Result<Unit, Error> {
        return db.Exec("INSERT INTO users ...", entity)?
    }
    override fun Delete(id: String): Result<Unit, Error> {
        return db.Exec("DELETE FROM users WHERE id = ?", id)?
    }
}
```

---

## 15. Ranges

```kotlin
// Inclusive range
for (i in 0..5) {
    println(i)  // 0, 1, 2, 3, 4, 5
}

// Exclusive upper bound
for (i in 0 until 5) {
    println(i)  // 0, 1, 2, 3, 4
}

// Step
for (i in 0..10 step 2) {
    println(i)  // 0, 2, 4, 6, 8, 10
}

// Downward
for (i in 10 downTo 0) {
    println(i)  // 10, 9, 8, ... 0
}

// Range check
val age = 25
if (age in 18..65) {
    println("Working age")
}
```

---

## 16. Destructuring Declarations

```kotlin
// Data class destructuring
data class Point(val x: Float, val y: Float)
val (x, y) = Point(3.0, 4.0)

// Pair destructuring
val (name, score) = Pair("Alice", 95)

// In for loops
val scores = mapOf("Alice" to 95, "Bob" to 87)
for ((name, score) in scores) {
    println("$name: $score")
}

// Nested destructuring
data class Line(val start: Point, val end: Point)
val line = Line(Point(0.0, 0.0), Point(10.0, 10.0))
val (start, end) = line
val (x1, y1) = start

// Ignoring components with _
val (_, age) = User("Alice", 30)  // only need the age

// Option destructuring
when (val opt = FindUser(42)) {
    is Some -> {
        val (name, age) = opt.value
        println("$name is $age")
    }
    is None -> println("Not found")
}
```

---

## 17. Operator Overloading

```kotlin
// Convention-based — implement named methods
data class Vec2(val x: Float, val y: Float) {
    // + operator
    operator fun plus(other: Vec2): Vec2 = Vec2(x + other.x, y + other.y)

    // - operator
    operator fun minus(other: Vec2): Vec2 = Vec2(x - other.x, y - other.y)

    // * operator (scalar)
    operator fun times(scalar: Float): Vec2 = Vec2(x * scalar, y * scalar)

    // [] operator (indexed access)
    operator fun get(index: Int): Float = when (index) {
        0 -> x
        1 -> y
        else -> throw IndexOutOfBoundsException()
    }

    // == operator (auto-generated by data class, but can override)
    // equals() is already generated
}

// Usage
val a = Vec2(1.0, 2.0)
val b = Vec2(3.0, 4.0)
val c = a + b           // Vec2(4.0, 6.0)
val d = a * 2.0         // Vec2(2.0, 4.0)
val xVal = a[0]         // 1.0
```

---

## 18. Properties (Custom Getters / Setters)

```kotlin
data class Temperature(val celsius: Float) {
    // Computed property (read-only)
    val fahrenheit: Float
        get() = celsius * 9.0 / 5.0 + 32.0

    val kelvin: Float
        get() = celsius + 273.15
}

val t = Temperature(100.0)
println(t.fahrenheit)  // 212.0
println(t.kelvin)      // 373.15

// Mutable property with custom setter
class BoundedCounter {
    var value: Int = 0
        set(newValue) {
            field = newValue.coerceIn(0, 100)
        }
}

val counter = BoundedCounter()
counter.value = 150
println(counter.value)  // 100 (clamped)
```

---

## 19. Go Concurrency (Preserved)

```kotlin
// Goroutines
go {
    println("Running in a goroutine")
}

// Channels
val ch = Channel<String>(10)  // buffered channel

go {
    ch.send("hello")
    ch.send("world")
    ch.close()
}

for (msg in ch) {
    println(msg)
}

// Unbuffered channel
val signal = Channel<Unit>()

go {
    DoWork()
    signal.send(Unit)
}
signal.receive()  // wait for completion

// Select
val ch1 = Channel<String>()
val ch2 = Channel<Int>()

select {
    ch1.onReceive { msg -> println("Got string: $msg") }
    ch2.onReceive { num -> println("Got number: $num") }
}

// Defer
fun ProcessFile(path: String): Result<Unit, Error> {
    val file = OpenFile(path)?
    defer { file.Close() }

    val data = file.ReadAll()?
    Process(data)
    return Ok(Unit)
}
```

---

## 20. Go Interop

```kotlin
// Import Go packages directly
import "fmt"
import "net/http"
import "encoding/json"
import "github.com/gorilla/mux"

// Use Go types — auto-wrapped with Bak safety
fun StartServer(port: Int) {
    val router = mux.NewRouter()

    router.HandleFunc("/users/{id}") { w, r ->
        val vars = mux.Vars(r)
        val id: String? = vars["id"]  // nullable — might not exist

        when (val user = FindUser(id ?: "")) {
            is Ok  -> json.NewEncoder(w).Encode(user.value)
            is Err -> {
                w.WriteHeader(404)
                fmt.Fprintf(w, "Not found: %s", user.error)
            }
        }
    }

    fmt.Printf("Listening on :%d\n", port)
    http.ListenAndServe(":$port", router)
}

// Go error returns auto-wrap to Result
fun ReadConfig(): Result<Config, Error> {
    val data = os.ReadFile("config.json")?    // Go ([]byte, error) → Result
    val config = json.Unmarshal(data)?         // same auto-wrapping
    return Ok(config)
}

// Bak-generated code is importable from Go
// This Bak function:
fun Add(a: Int, b: Int): Int = a + b

// Generates this Go:
// func Add(a int, b int) int { return a + b }
// Any Go code can import and call it
```

---

## 21. Visibility (Go Conventions)

```kotlin
// Uppercase = exported (public) — visible outside the package
fun ProcessOrder(order: Order): Result<Receipt, Error> { /* ... */ }
data class User(val Name: String, val Email: String)

// Lowercase = unexported (private) — package-internal only
fun validateEmail(email: String): Boolean { /* ... */ }
data class cache(val entries: Map<String, Any>)

// Traits follow the same rule
trait Exportable {       // exported trait
    fun Export(): ByteArray
}

trait internal {         // unexported trait
    fun setup()
}
```

---

## 22. Scope Functions (Standard Library)

```kotlin
// let — transform a value
val length = "hello".let { it.length }  // 5

// also — side effects, returns original
val user = CreateUser("Alice").also {
    println("Created user: ${it.name}")
    LogAudit("user_created", it.id)
}

// apply — configure an object, returns the object
val config = Config().apply {
    timeout = 30
    retries = 3
    baseUrl = "https://api.example.com"
}

// with — operate on an object
val description = with(user) {
    "$name is $age years old and lives in $city"
}

// run — like let but with receiver
val result = service.run {
    Connect()
    FetchData()
}
```

---

## 23. Full Example: A Complete Bak Program

```kotlin
import "net/http"
import "encoding/json"
import "log"

// --- Domain Model ---

data class Todo(
    val id: String,
    val title: String,
    val completed: Boolean = false
)

sealed class TodoError {
    data class NotFound(val id: String) : TodoError()
    data class ValidationFailed(val message: String) : TodoError()
    data class StorageError(val cause: Error) : TodoError()
}

// --- Repository ---

trait TodoRepository {
    fun FindById(id: String): Result<Todo, TodoError>
    fun FindAll(): Result<List<Todo>, TodoError>
    fun Save(todo: Todo): Result<Todo, TodoError>
    fun Delete(id: String): Result<Unit, TodoError>
}

// --- Service ---

data class TodoService(val repo: TodoRepository) {

    fun CreateTodo(title: String): Result<Todo, TodoError> {
        if (title.isBlank()) {
            return Err(TodoError.ValidationFailed("Title cannot be blank"))
        }
        val todo = Todo(
            id = GenerateId(),
            title = title.trim()
        )
        return repo.Save(todo)
    }

    fun CompleteTodo(id: String): Result<Todo, TodoError> {
        val todo = repo.FindById(id)?
        val updated = todo.copy(completed = true)
        return repo.Save(updated)
    }

    fun ListActive(): Result<List<Todo>, TodoError> {
        val all = repo.FindAll()?
        return Ok(all.filter { !it.completed })
    }
}

// --- HTTP Handler ---

fun TodoHandler(service: TodoService): http.Handler {
    val mux = http.NewServeMux()

    mux.HandleFunc("GET /todos") { w, r ->
        when (val result = service.ListActive()) {
            is Ok -> {
                w.Header().Set("Content-Type", "application/json")
                json.NewEncoder(w).Encode(result.value)
            }
            is Err -> {
                w.WriteHeader(500)
                log.Printf("Error listing todos: %v", result.error)
            }
        }
    }

    mux.HandleFunc("POST /todos") { w, r ->
        val title = r.FormValue("title")
        when (val result = service.CreateTodo(title)) {
            is Ok -> {
                w.WriteHeader(201)
                json.NewEncoder(w).Encode(result.value)
            }
            is Err -> when (result.error) {
                is TodoError.ValidationFailed -> {
                    w.WriteHeader(400)
                    fmt.Fprintf(w, result.error.message)
                }
                else -> w.WriteHeader(500)
            }
        }
    }

    return mux
}

// --- Main ---

fun main() {
    val repo = InMemoryTodoRepo()
    val service = TodoService(repo)
    val handler = TodoHandler(service)

    val ch = Channel<Error>()

    go {
        log.Println("Starting server on :8080")
        ch.send(http.ListenAndServe(":8080", handler))
    }

    val err = ch.receive()
    log.Fatal(err)
}
```
