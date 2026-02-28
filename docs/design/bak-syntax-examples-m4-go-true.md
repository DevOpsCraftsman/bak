# Bak Language — Syntax Examples (M4: `:=` + `var` Go-True)

Complete reference using the **M4 syntax**: `:=` immutable by default, `var` for
mutability (same meaning as Go), `fn` keyword, no colon in type annotations.

---

## 1. Variables and Type Inference

```go
// Immutable (default) — no keyword needed
x := 42
name := "Alice"
pi := 3.14
flag := true

// With explicit type
x Int := 42
name String := "Alice"

// Mutable — var keyword
var counter := 0
counter = counter + 1

// Uninitialized — needs explicit type (Go's zero value)
var count Int              // 0
var label String           // ""
var user User?             // null
```

---

## 2. Null Safety

```go
// Non-null by default
name := "Alice"

// Nullable — must use T?
maybeName String? := null
anotherName String? := "Bob"

// Safe call operator
len Int? := maybeName?.length

// Elvis operator
displayName := maybeName ?: "Anonymous"

// Chained safe calls
city String? := user?.address?.city

// Non-null assertion — throws if null
forced String := maybeName!!

// Safe call + elvis
upperName := maybeName?.uppercase() ?: "UNKNOWN"
```

---

## 3. Option Type (Some / None)

```go
// Explicit Option type
present Option<Int> := Some(42)
absent Option<Int> := None

// Pattern matching
match present {
    is Some(value) -> println("Got: $value")
    is None -> println("Nothing here")
}

// FP composition
doubled := present.map { it * 2 }           // Some(84)
nothing := absent.map { it * 2 }            // None
trimmed := present.filter { it > 10 }       // Some(42)
chained := present.flatMap { lookupName(it) }

// Unwrap with default
value := present.unwrapOr(0)                // 42
fallback := absent.unwrapOr(0)              // 0

// Bridge: T? → Option<T>
name String? := "Alice"
opt Option<String> := name.toOption()        // Some("Alice")
opt2 Option<String> := null.toOption()       // None

// Bridge: Option<T> → T?
nullable String? := opt.toNullable()         // "Alice"
nullable2 String? := None.toNullable()       // null
```

---

## 4. Result Type and Error Handling

```go
// Result<T, E> — Ok or Err
fn readFile(path String) Result<String, IOError> {
    // ...
}

// Pattern matching on Result
match readFile("config.yaml") {
    is Ok(content) -> println("Content: $content")
    is Err(e) -> println("Failed: ${e.message}")
}

// ? operator — early return on error
fn loadConfig(path String) Result<Config, AppError> {
    content := readFile(path)?
    parsed := parseYaml(content)?
    validated := validate(parsed)?
    return Ok(validated)
}

// Result API
result Result<User, DBError> := findUser(42)

result.map { it.name }                       // Result<String, DBError>
result.flatMap { validate(it) }              // Result<ValidUser, DBError>
result.mapError { AppError.from(it) }        // Result<User, AppError>
result.recover { defaultUser }               // User
result.fold(
    onOk  = { println("Found: ${it.name}") },
    onErr = { println("Failed: ${it.message}") }
)
result.getOrElse { defaultUser }             // User

// Go interop — (T, error) auto-wraps to Result
import "os"

fn readGoFile(path String) Result<[]byte, Error> {
    return os.ReadFile(path)
}
```

---

## 4b. Either Type (Standard Library)

```go
import "bak/either"

fn parseInput(raw String) Either<Int, String> {
    asInt := raw.toIntOrNull()
    return if asInt != null { Either.left(asInt) } else { Either.right(raw) }
}

parsed Either<Int, String> := parseInput("hello")

parsed.fold(
    ifLeft  = { println("Got number: $it") },
    ifRight = { println("Got text: $it") }
)
parsed.map { it.uppercase() }
parsed.mapLeft { it * 2 }
parsed.swap()

// Convert between Result and Either
asResult Result<String, Int> := parsed.toResult()
asEither Either<DBError, User> := result.toEither()
```

---

## 5. Functions

```go
// Standard function
fn add(a Int, b Int) Int {
    return a + b
}

// Expression body
fn add(a Int, b Int) Int = a + b

// Return type inference
fn multiply(a Int, b Int) = a * b

// Default parameter values
fn greet(name String, greeting String = "Hello") String {
    return "$greeting, $name!"
}

// Calling with defaults
msg1 := greet("Alice")                     // "Hello, Alice!"
msg2 := greet("Bob", "Hi")                // "Hi, Bob!"

// Named arguments
msg3 := greet(greeting = "Hey", name = "Charlie")

// Unit return (no return type)
fn log(message String) {
    println(message)
}
```

---

## 6. String Interpolation

```go
name := "Alice"
age := 30

println("Hello, $name")
println("In 5 years: ${age + 5}")
println("Uppercase: ${name.uppercase()}")

// Multi-line strings
json := """
    {
        "name": "$name",
        "age": $age
    }
"""
```

---

## 7. Data Classes

```go
// Auto-generates: Equal(), String(), Copy(), hash
data class User(name String, age Int)

alice := User("Alice", 30)
bob := User("Bob", 25)

println(alice)                              // User(name=Alice, age=30)
println(alice == User("Alice", 30))         // true

// Copy with modifications
olderAlice := alice.copy(age = 31)

// Destructuring
(name, age) := alice
println("$name is $age years old")

// Nested data classes
data class Address(street String, city String, zip String)
data class Person(name String, address Address)

person := Person("Alice", Address("123 Main St", "Springfield", "62701"))
```

---

## 8. Interfaces (with Default Implementations)

```go
// Basic interface
interface Drawable {
    fn draw()
    fn boundingBox() Rect
}

// Interface with default implementation
interface Serializable {
    fn serialize() []byte
    fn contentType() String = "application/json"
    fn serializeToString() String = String(serialize())
}

// Implementing an interface
data class Circle(x Float, y Float, radius Float) : Drawable {
    override fn draw() {
        // drawing logic
    }
    override fn boundingBox() Rect {
        return Rect(x - radius, y - radius, radius * 2, radius * 2)
    }
}

// Multiple interfaces
data class Widget(id String) : Drawable, Serializable {
    override fn draw() { /* ... */ }
    override fn boundingBox() = Rect(0, 0, 100, 50)
    override fn serialize() = id.toByteArray()
}

// Interface as parameter type
fn render(item Drawable) {
    item.draw()
}

// Interface bounds on generics
fn <T : Serializable> save(item T) {
    bytes := item.serialize()
    writeToFile(bytes)
}
```

---

## 9. Sealed Classes and Pattern Matching

```go
sealed class Shape {
    data class Circle(radius Float)
    data class Rectangle(width Float, height Float)
    data class Triangle(base Float, height Float)
}

fn area(shape Shape) Float = match shape {
    is Shape.Circle(r) -> 3.14159 * r * r
    is Shape.Rectangle(w, h) -> w * h
    is Shape.Triangle(b, h) -> 0.5 * b * h
}

// Sealed for domain modeling
sealed class AuthResult {
    data class Success(user User, token String)
    data class Failure(reason String)
    data class TwoFactorRequired(challengeId String)
}

fn handleAuth(result AuthResult) {
    match result {
        is AuthResult.Success(user, token) -> {
            setSession(user, token)
            redirect("/dashboard")
        }
        is AuthResult.Failure(reason) -> {
            showError(reason)
        }
        is AuthResult.TwoFactorRequired(id) -> {
            show2FAPrompt(id)
        }
    }
}

// when — for conditional branching (no subject)
fn classify(value Int) String = when {
    value < 0   -> "negative"
    value == 0  -> "zero"
    value < 100 -> "small"
    else        -> "large"
}
```

---

## 10. Smart Casts

```go
fn describe(x Any) String {
    if x is String {
        return "String of length ${x.length}"
    }
    if x is Int {
        return "Integer: ${x * 2}"
    }
    return "Unknown"
}

fn process(input Any) {
    match input {
        is String -> println(input.uppercase())
        is Int    -> println(input + 1)
        is List   -> println(input.size)
    }
}

fn safeProcess(value String?) {
    if value != null {
        println(value.length)
    }
}
```

---

## 11. Extension Functions

```go
fn String.isPalindrome() Bool {
    cleaned := this.lowercase().filter { it.isLetter() }
    return cleaned == cleaned.reversed()
}

result := "racecar".isPalindrome()       // true

fn Int.isEven() Bool = this % 2 == 0

check := 42.isEven()                     // true

// Extension on Go stdlib types
fn http.Request.bearerToken() String? {
    header := this.Header.Get("Authorization")
    return if header.startsWith("Bearer ") { header.substring(7) } else { null }
}

// Extension with generics
fn <T> List<T>.secondOrNull() T? {
    return if this.size >= 2 { this[1] } else { null }
}
```

---

## 12. Lambdas and Higher-Order Functions

```go
double := { x Int -> x * 2 }
sum := { a Int, b Int -> a + b }

fn apply(value Int, transform (Int) -> Int) Int {
    return transform(value)
}
result := apply(5, double)               // 10

// Trailing lambda
numbers := listOf(1, 2, 3, 4, 5)

doubled := numbers.map { it * 2 }        // [2, 4, 6, 8, 10]
evens := numbers.filter { it % 2 == 0 }  // [2, 4]
total := numbers.reduce { acc, n -> acc + n }  // 15

// Chaining
result := numbers
    .filter { it > 2 }
    .map { it * 10 }
    .reduce { acc, n -> acc + n }         // 120

// forEach
numbers.forEach { println(it) }

// Function references
fn isPositive(n Int) Bool = n > 0
positives := numbers.filter(::isPositive)
```

---

## 13. Enums

```go
enum class Direction {
    North, South, East, West
}

enum class Color(hex String) {
    Red("#FF0000"),
    Green("#00FF00"),
    Blue("#0000FF")
}

dir := Direction.North
color := Color.Red
println(color.hex)                        // #FF0000

fn arrow(dir Direction) String = match dir {
    Direction.North -> "↑"
    Direction.South -> "↓"
    Direction.East  -> "→"
    Direction.West  -> "←"
}
```

---

## 14. Generics

```go
fn <T> first(list List<T>) Option<T> {
    return if list.isEmpty() { None } else { Some(list[0]) }
}

fn <T : Comparable<T>> max(a T, b T) T = if a > b { a } else { b }

data class Pair<A, B>(first A, second B)

pair := Pair("age", 30)
(key, value) := pair

interface Repository<T> {
    fn findById(id String) Result<T, Error>
    fn save(entity T) Result<Unit, Error>
    fn delete(id String) Result<Unit, Error>
}

data class UserRepo(db Database) : Repository<User> {
    override fn findById(id String) Result<User, Error> {
        return db.query("SELECT * FROM users WHERE id = ?", id)
    }
    override fn save(entity User) Result<Unit, Error> {
        return db.exec("INSERT INTO users ...", entity)
    }
    override fn delete(id String) Result<Unit, Error> {
        return db.exec("DELETE FROM users WHERE id = ?", id)
    }
}
```

---

## 15. Ranges

```go
for i in 0..5 {
    println(i)                            // 0, 1, 2, 3, 4, 5
}

for i in 0 until 5 {
    println(i)                            // 0, 1, 2, 3, 4
}

for i in 0..10 step 2 {
    println(i)                            // 0, 2, 4, 6, 8, 10
}

for i in 10 downTo 0 {
    println(i)                            // 10, 9, 8, ... 0
}

age := 25
if age in 18..65 {
    println("Working age")
}
```

---

## 16. Destructuring Declarations

```go
data class Point(x Float, y Float)
(x, y) := Point(3.0, 4.0)

(name, score) := Pair("Alice", 95)

scores := mapOf("Alice" to 95, "Bob" to 87)
for (name, score) in scores {
    println("$name: $score")
}

// Ignoring components
(_, age) := User("Alice", 30)

// Option destructuring
match findUser(42) {
    is Some(user) -> {
        (name, age) := user
        println("$name is $age")
    }
    is None -> println("Not found")
}
```

---

## 17. Operator Overloading

```go
data class Vec2(x Float, y Float) {
    operator fn plus(other Vec2) Vec2 = Vec2(x + other.x, y + other.y)
    operator fn minus(other Vec2) Vec2 = Vec2(x - other.x, y - other.y)
    operator fn times(scalar Float) Vec2 = Vec2(x * scalar, y * scalar)
    operator fn get(index Int) Float = match index {
        0 -> x
        1 -> y
        else -> throw IndexOutOfBoundsException()
    }
}

a := Vec2(1.0, 2.0)
b := Vec2(3.0, 4.0)
c := a + b                               // Vec2(4.0, 6.0)
d := a * 2.0                             // Vec2(2.0, 4.0)
xVal := a[0]                             // 1.0
```

---

## 18. Properties (Custom Getters / Setters)

```go
data class Temperature(celsius Float) {
    fahrenheit Float
        get() = celsius * 9.0 / 5.0 + 32.0

    kelvin Float
        get() = celsius + 273.15
}

t := Temperature(100.0)
println(t.fahrenheit)                     // 212.0
println(t.kelvin)                         // 373.15

class BoundedCounter {
    var value Int = 0
        set(newValue) {
            field = newValue.coerceIn(0, 100)
        }
}

counter := BoundedCounter()
counter.value = 150
println(counter.value)                    // 100 (clamped)
```

---

## 19. Go Concurrency (Preserved)

```go
go {
    println("Running in a goroutine")
}

ch := Channel<String>(10)

go {
    ch.send("hello")
    ch.send("world")
    ch.close()
}

for msg in ch {
    println(msg)
}

// Select
ch1 := Channel<String>()
ch2 := Channel<Int>()

select {
    ch1.onReceive { msg -> println("Got string: $msg") }
    ch2.onReceive { num -> println("Got number: $num") }
}

// Defer
fn processFile(path String) Result<Unit, Error> {
    file := openFile(path)?
    defer { file.close() }

    data := file.readAll()?
    process(data)
    return Ok(Unit)
}
```

---

## 20. Go Interop

```go
import "fmt"
import "net/http"
import "encoding/json"
import "github.com/gorilla/mux"

fn startServer(port Int) {
    router := mux.NewRouter()

    router.HandleFunc("/users/{id}") { w, r ->
        vars := mux.Vars(r)
        id String? := vars["id"]

        match findUser(id ?: "") {
            is Ok(user) -> json.NewEncoder(w).Encode(user)
            is Err(e) -> {
                w.WriteHeader(404)
                fmt.Fprintf(w, "Not found: %s", e)
            }
        }
    }

    fmt.Printf("Listening on :%d\n", port)
    http.ListenAndServe(":$port", router)
}

// Go error returns auto-wrap to Result
fn readConfig() Result<Config, Error> {
    data := os.ReadFile("config.json")?
    config := json.Unmarshal(data)?
    return Ok(config)
}

// Bak functions are callable from Go
fn add(a Int, b Int) Int = a + b
// Generates: func Add(a int, b int) int { return a + b }
```

---

## 21. Visibility (Go Conventions)

```go
// Uppercase = exported (public)
fn ProcessOrder(order Order) Result<Receipt, Error> { /* ... */ }
data class User(Name String, Email String)

// Lowercase = unexported (private)
fn validateEmail(email String) Bool { /* ... */ }
data class cache(entries Map<String, Any>)

interface Exportable {
    fn Export() []byte
}

interface internal {
    fn setup()
}
```

---

## 22. Scope Functions (Standard Library)

```go
length := "hello".let { it.length }       // 5

user := createUser("Alice").also {
    println("Created user: ${it.name}")
    logAudit("user_created", it.id)
}

config := Config().apply {
    timeout = 30
    retries = 3
    baseUrl = "https://api.example.com"
}

description := with(user) {
    "$name is $age years old and lives in $city"
}

result := service.run {
    connect()
    fetchData()
}
```

---

## 23. Full Example: A Complete Bak Program

```go
import "net/http"
import "encoding/json"
import "log"

// --- Domain Model ---

data class Todo(
    id String,
    title String,
    var completed Bool = false
)

sealed class TodoError {
    data class NotFound(id String)
    data class ValidationFailed(message String)
    data class StorageError(cause Error)
}

// --- Repository ---

interface TodoRepository {
    fn findById(id String) Result<Todo, TodoError>
    fn findAll() Result<List<Todo>, TodoError>
    fn save(todo Todo) Result<Todo, TodoError>
    fn delete(id String) Result<Unit, TodoError>
}

// --- Service ---

data class TodoService(repo TodoRepository) {

    fn createTodo(title String) Result<Todo, TodoError> {
        if title.isBlank() {
            return Err(TodoError.ValidationFailed("Title cannot be blank"))
        }
        todo := Todo(
            id = generateId(),
            title = title.trim()
        )
        return repo.save(todo)
    }

    fn completeTodo(id String) Result<Todo, TodoError> {
        todo := repo.findById(id)?
        updated := todo.copy(completed = true)
        return repo.save(updated)
    }

    fn listActive() Result<List<Todo>, TodoError> {
        all := repo.findAll()?
        return Ok(all.filter { !it.completed })
    }
}

// --- HTTP Handler ---

fn todoHandler(service TodoService) http.Handler {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /todos") { w, r ->
        match service.listActive() {
            is Ok(todos) -> {
                w.Header().Set("Content-Type", "application/json")
                json.NewEncoder(w).Encode(todos)
            }
            is Err(e) -> {
                w.WriteHeader(500)
                log.Printf("Error listing todos: %v", e)
            }
        }
    }

    mux.HandleFunc("POST /todos") { w, r ->
        title := r.FormValue("title")
        match service.createTodo(title) {
            is Ok(todo) -> {
                w.WriteHeader(201)
                json.NewEncoder(w).Encode(todo)
            }
            is Err(e) -> match e {
                is TodoError.ValidationFailed(msg) -> {
                    w.WriteHeader(400)
                    fmt.Fprintf(w, msg)
                }
                else -> w.WriteHeader(500)
            }
        }
    }

    return mux
}

// --- Main ---

fn main() {
    repo := InMemoryTodoRepo()
    service := TodoService(repo)
    handler := todoHandler(service)

    var ch := Channel<Error>()

    go {
        log.Println("Starting server on :8080")
        ch.send(http.ListenAndServe(":8080", handler))
    }

    err := ch.receive()
    log.Fatal(err)
}
```
