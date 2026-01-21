# Kira-Toolkit Coding Standards

> Hardened Kira development standards for safe, consistent, and maintainable functional code.

These standards define unambiguous, enforceable rules for writing Kira code. They are designed to work with AI-assisted development (Claude Code) and provide clear guidance for both humans and AI to follow.

**Version**: 0.1.0 (Initial Release)

---

## Table of Contents

1. [Language Standards](#1-language-standards)
2. [Naming Conventions](#2-naming-conventions)
3. [Purity and Effects](#3-purity-and-effects)
4. [Pattern Matching](#4-pattern-matching)
5. [Algebraic Data Types](#5-algebraic-data-types)
6. [Higher-Order Functions](#6-higher-order-functions)
7. [Error Handling](#7-error-handling)
8. [API Design](#8-api-design)
9. [Security](#9-security)
10. [Code Organization](#10-code-organization)
11. [Documentation](#11-documentation)
12. [Testing](#12-testing)
13. [Logging](#13-logging)

---

## 1. Language Standards

### 1.1 Kira Version

Target current Kira as defined in the language specification.

**Core Features:**
- Pure functions by default
- Explicit effect tracking with `effect fn`
- Algebraic data types (sum and product types)
- Pattern matching with exhaustiveness checking
- Higher-order functions
- Explicit types everywhere (no inference)

### 1.2 Core Principles

1. **Pure by default** — All functions are pure unless marked `effect`
2. **Explicit types everywhere** — No type inference
3. **Explicit effects** — IO and state visible in types
4. **One obvious way** — Single syntax for each construct
5. **Strict evaluation** — No lazy surprises
6. **No implicit currying** — Call sites look like definitions
7. **AI-first design** — Optimized for AI code generation clarity

### 1.3 File Extension

Kira source files use the `.ki` extension.

---

## 2. Naming Conventions

### 2.1 Case Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Types (sum, product) | `PascalCase` | `Option`, `UserData`, `HttpError` |
| Functions | `snake_case` | `create_user`, `parse_int` |
| Variables | `snake_case` | `user_count`, `is_valid` |
| Record fields | `snake_case` | `user_name`, `created_at` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_USERS`, `DEFAULT_PORT` |
| Sum type variants | `PascalCase` | `Some`, `None`, `Ok`, `Err` |
| Modules | `snake_case` | `std.list`, `std.option` |
| Type parameters | Single uppercase or `PascalCase` | `T`, `Key`, `Value` |

### 2.2 Function Naming Patterns

| Pattern | Meaning | Example |
|---------|---------|---------|
| `create` | Construct a new value | `create_user(name)` |
| `from_*` | Convert from another type | `from_string(s)` |
| `to_*` | Convert to another type | `to_string(n)` |
| `parse_*` | Parse from string/data | `parse_int(s)` |
| `is_*` / `has_*` / `can_*` | Boolean query | `is_empty(list)` |
| `try_*` | May fail, returns Result | `try_parse(input)` |
| `*_or` | Fallback variant | `unwrap_or(default)` |
| `map_*` | Transform contained value | `map_err(f)` |
| `and_then` | Chain operations | `and_then(f)` |

### 2.3 Boolean Naming

Boolean variables and functions MUST read as true/false statements:

```kira
// GOOD
let is_valid: bool = true
let has_items: bool = std.list.length(items) > 0
let can_proceed: fn(State) -> bool = fn(s: State) -> bool { ... }

// BAD
let valid: bool = true        // Ambiguous
let items: bool = true        // Noun, not predicate
let proceed: fn() -> bool     // Verb, not query
```

Preferred prefixes: `is_`, `has_`, `can_`, `should_`, `was_`, `will_`

### 2.4 Acronyms and Abbreviations

Treat acronyms as words for casing:

```kira
// GOOD
type HttpClient = { ... }
let parse_json: fn(string) -> Result[Json, ParseError] = ...
let html_content: string = fetch_page(url)

// BAD
type HTTPClient = { ... }   // Use Http, not HTTP
let parseJSON: fn(...) = ...  // Use Json, not JSON
```

Acceptable abbreviations: `len`, `str`, `num`, `max`, `min`, `err`, `msg`, `ctx`

---

## 3. Purity and Effects

Kira's core feature is tracking purity at the type level.

### 3.1 Pure Functions (Default)

Functions without `effect` keyword are pure:

```kira
// Pure function - cannot perform IO, mutation, or other effects
let add: fn(i32, i32) -> i32 = fn(a: i32, b: i32) -> i32 {
    return a + b
}

let factorial: fn(i32) -> i32 = fn(n: i32) -> i32 {
    if n <= 1 {
        return 1
    }
    return n * factorial(n - 1)
}
```

**P1**: Pure functions cannot:
- Perform IO (print, read files, network)
- Mutate global state
- Generate random numbers
- Get current time
- Call effectful functions

### 3.2 Effect Functions

Use `effect fn` for functions that perform side effects:

```kira
// Effect function - can perform IO
effect fn print_greeting(name: string) -> IO[void] {
    std.io.println("Hello, {name}!")
    return
}

// Effect function with error handling
effect fn read_file(path: string) -> IO[Result[string, IoError]] {
    return std.fs.read_file(path)
}

// Main is always an effect function
effect fn main() -> IO[void] {
    let result: i32 = factorial(5)   // OK: calling pure from effect
    print_greeting("World")           // OK: calling effect from effect
    return
}
```

**P2**: Effect functions can call both pure and effectful code.

**P3**: Pure functions CANNOT call effectful code (compile error).

### 3.3 Effect Types

| Effect Type | Meaning |
|-------------|---------|
| `IO[T]` | Function performs IO, returns T |
| `IO[Result[T, E]]` | IO that may fail with error E |
| `IO[Option[T]]` | IO that may return nothing |

### 3.4 Effect Propagation

```kira
// Pure code cannot call effectful code - this is a compile error:
// let bad: fn(i32) -> i32 = fn(x: i32) -> i32 {
//     std.io.println("hello")  // ERROR: pure function cannot do IO
//     return x
// }

// Effectful code can call both
effect fn process(x: i32) -> IO[i32] {
    let doubled: i32 = x * 2         // OK: pure computation
    std.io.println("processing")      // OK: IO in IO context
    return doubled
}
```

### 3.5 Purity Benefits

**P4**: Pure functions enable:
- Safe parallelization
- Memoization
- Equational reasoning
- Easy testing (no mocks needed)
- Refactoring confidence

---

## 4. Pattern Matching

### 4.1 Match Statement

Match is a statement that assigns to a variable. All cases MUST be handled.

```kira
let describe: fn(Option[i32]) -> string = fn(opt: Option[i32]) -> string {
    var result: string
    match opt {
        Some(n) => { result = "has value: {n}" }
        None => { result = "empty" }
    }
    return result
}
```

**PM1**: Pattern matches MUST be exhaustive - all possible values must be covered.

### 4.2 Pattern Types

```kira
// Literal patterns
match value {
    0 => { result = "zero" }
    1 => { result = "one" }
    _ => { result = "other" }
}

// Constructor patterns (sum types)
match option {
    Some(x) => { result = x }
    None => { result = 0 }
}

// Record patterns (product types)
match point {
    Point { x: 0, y: 0 } => { result = "origin" }
    Point { x: 0, y: y } => { result = "on y-axis at {y}" }
    Point { x: x, y: 0 } => { result = "on x-axis at {x}" }
    Point { x: x, y: y } => { result = "at ({x}, {y})" }
}

// Tuple patterns
match pair {
    (0, 0) => { result = "origin" }
    (x, 0) => { result = "x = {x}" }
    (0, y) => { result = "y = {y}" }
    (x, y) => { result = "({x}, {y})" }
}

// Or patterns
match value {
    1 | 2 | 3 => { result = "small" }
    _ => { result = "other" }
}
```

### 4.3 Guard Patterns

```kira
match value {
    n if n < 0 => { result = "negative" }
    n if n > 100 => { result = "large" }
    n => { result = "normal: {n}" }
}
```

**PM2**: Place more specific patterns before general ones.

### 4.4 Destructuring in Let

```kira
// Destructure records
let p: Point = Point { x: 3.0, y: 4.0 }
let Point { x: px, y: py }: Point = p

// Destructure tuples
let pair: (i32, string) = (42, "hello")
let (n, s): (i32, string) = pair
```

### 4.5 Nested Patterns

```kira
match nested {
    Some(Ok(value)) => { result = value }
    Some(Err(e)) => { result = handle_error(e) }
    None => { result = default_value }
}
```

**PM3**: Prefer nested patterns over nested match statements.

---

## 5. Algebraic Data Types

### 5.1 Sum Types (Tagged Unions)

Sum types represent "one of" several variants:

```kira
// Basic sum type
type Option[T] =
    | Some(T)
    | None

// Sum type with multiple data
type Result[T, E] =
    | Ok(T)
    | Err(E)

// Sum type with named fields in variants
type HttpError =
    | NotFound(string)
    | Unauthorized { reason: string }
    | ServerError { code: i32, message: string }

// Recursive sum type
type List[T] =
    | Cons(T, List[T])
    | Nil
```

**ADT1**: Use sum types for values that can be one of several variants.

### 5.2 Product Types (Records)

Product types represent "all of" several fields:

```kira
// Basic product type
type Point = {
    x: f64,
    y: f64
}

// Product type with multiple fields
type User = {
    id: i64,
    name: string,
    email: string,
    created_at: i64
}

// Creating values
let origin: Point = Point { x: 0.0, y: 0.0 }
let user: User = User {
    id: 1,
    name: "Alice",
    email: "alice@example.com",
    created_at: 1234567890
}
```

**ADT2**: Use product types for values that have multiple named fields.

### 5.3 Generic Types

```kira
// Generic sum type
type Either[A, B] =
    | Left(A)
    | Right(B)

// Generic product type
type Pair[A, B] = {
    first: A,
    second: B
}

// Usage with explicit type parameters
let pair: Pair[i32, string] = Pair { first: 42, second: "hello" }
let either: Either[string, i32] = Left("error")
```

### 5.4 Type Aliases

```kira
// Simple alias
type UserId = i64
type Email = string

// Generic alias
type StringResult[T] = Result[T, string]
type IntList = List[i32]
```

### 5.5 ADT Design Principles

**ADT3**: Make illegal states unrepresentable:

```kira
// BAD: Allows invalid states
type User = {
    name: string,
    email: Option[string],
    email_verified: bool  // Can be true with no email!
}

// GOOD: Invalid states impossible
type EmailStatus =
    | Unverified(string)
    | Verified(string)
    | NoEmail

type User = {
    name: string,
    email: EmailStatus
}
```

**ADT4**: Prefer many specific types over few general types:

```kira
// BAD: Overly general
type Event = {
    event_type: string,
    data: string
}

// GOOD: Specific types
type Event =
    | UserCreated { user_id: i64, name: string }
    | UserDeleted { user_id: i64 }
    | LoginAttempt { user_id: i64, success: bool }
```

---

## 6. Higher-Order Functions

### 6.1 Functions as Values

Functions are first-class values in Kira:

```kira
// Function binding
let add: fn(i32, i32) -> i32 = fn(a: i32, b: i32) -> i32 {
    return a + b
}

// Passing functions as arguments
let apply: fn(fn(i32, i32) -> i32, i32, i32) -> i32 =
    fn(f: fn(i32, i32) -> i32, a: i32, b: i32) -> i32 {
        return f(a, b)
    }

let result: i32 = apply(add, 5, 3)  // 8
```

### 6.2 Standard Higher-Order Functions

**HO1**: Use standard library functions for common operations:

```kira
import std.list.{ map, filter, fold }

let numbers: List[i32] = Cons(1, Cons(2, Cons(3, Cons(4, Cons(5, Nil)))))

// Map: transform each element
let doubled: List[i32] = map[i32, i32](
    numbers,
    fn(x: i32) -> i32 { return x * 2 }
)

// Filter: keep elements matching predicate
let evens: List[i32] = filter[i32](
    numbers,
    fn(x: i32) -> bool { return x % 2 == 0 }
)

// Fold: reduce to single value
let sum: i32 = fold[i32, i32](
    numbers,
    0,
    fn(acc: i32, x: i32) -> i32 { return acc + x }
)
```

### 6.3 Closures

Closures capture their environment:

```kira
// Closure capturing 'n' from environment
let make_adder: fn(i32) -> fn(i32) -> i32 = fn(n: i32) -> fn(i32) -> i32 {
    return fn(x: i32) -> i32 { return x + n }
}

let add_five: fn(i32) -> i32 = make_adder(5)
let result: i32 = add_five(10)  // 15
```

**HO2**: Closures capture environment immutably in pure functions.

### 6.4 Function Composition

```kira
// Explicit composition
let compose[A, B, C]: fn(fn(B) -> C, fn(A) -> B) -> fn(A) -> C =
    fn(f: fn(B) -> C, g: fn(A) -> B) -> fn(A) -> C {
        return fn(x: A) -> C {
            let intermediate: B = g(x)
            return f(intermediate)
        }
    }

// Usage
let add_one: fn(i32) -> i32 = fn(x: i32) -> i32 { return x + 1 }
let double: fn(i32) -> i32 = fn(x: i32) -> i32 { return x * 2 }

let add_then_double: fn(i32) -> i32 = compose[i32, i32, i32](double, add_one)
let result: i32 = add_then_double(5)  // (5 + 1) * 2 = 12
```

**HO3**: Prefer direct calls when composition would obscure intent:

```kira
// Often clearer than composition
let result: i32 = double(add_one(5))
```

### 6.5 Partial Application

```kira
// Kira doesn't have implicit currying
// Use explicit partial application

let partial_add: fn(i32) -> fn(i32) -> i32 = fn(a: i32) -> fn(i32) -> i32 {
    return fn(b: i32) -> i32 { return a + b }
}

let add_ten: fn(i32) -> i32 = partial_add(10)
let result: i32 = add_ten(5)  // 15
```

---

## 7. Error Handling

### 7.1 Result Type

All fallible operations MUST return `Result[T, E]`:

```kira
type Result[T, E] =
    | Ok(T)
    | Err(E)

let parse_int: fn(string) -> Result[i32, ParseError] =
    fn(s: string) -> Result[i32, ParseError] {
        // ... parsing logic
    }
```

**E1**: Use `Result[T, E]` for operations that can legitimately fail.

### 7.2 Option Type

Use `Option[T]` for values that may be absent (not errors):

```kira
type Option[T] =
    | Some(T)
    | None

let find: fn(List[i32], fn(i32) -> bool) -> Option[i32] =
    fn(list: List[i32], pred: fn(i32) -> bool) -> Option[i32] {
        match list {
            Nil => { return None }
            Cons(head, tail) => {
                if pred(head) {
                    return Some(head)
                }
                return find(tail, pred)
            }
        }
    }
```

**E2**: Use `Option[T]` when absence is normal, `Result[T, E]` when absence is an error.

### 7.3 Error Propagation

Use `?` to propagate errors:

```kira
effect fn load_config(path: string) -> IO[Result[Config, ConfigError]] {
    let content: string = std.fs.read_file(path)?    // Propagates IoError
    let parsed: Toml = parse_toml(content)?          // Propagates ParseError
    let config: Config = validate(parsed)?           // Propagates ValidationError
    return Ok(config)
}
```

**E3**: Use `?` for error propagation instead of explicit matching when appropriate.

### 7.4 Custom Error Types

Define meaningful error types for each module:

```kira
type ConfigError =
    | NotFound(string)
    | ParseError { line: i32, message: string }
    | ValidationError(string)
    | IoError(IoError)
```

**E4**: Include context in error variants (file path, line number, etc.).

### 7.5 Error Handling Patterns

```kira
// Pattern 1: Match for specific handling
var output: Config
match result {
    Ok(value) => { output = value }
    Err(ConfigError.NotFound(_)) => { output = use_defaults() }
    Err(e) => { return Err(e) }
}

// Pattern 2: Provide default with ??
let value: i32 = result ?? default_value

// Pattern 3: Map the success value
let mapped: Result[string, Error] = std.result.map[i32, Error, string](
    result,
    fn(n: i32) -> string { return n.to_string() }
)

// Pattern 4: Chain operations
let final_result: Result[Data, Error] = std.result.and_then[Step1, Error, Data](
    step_one(),
    fn(s1: Step1) -> Result[Data, Error] { return step_two(s1) }
)
```

### 7.6 Traps vs Errors

**E5**: Use traps (panic, assert) for bugs; use Result for expected failures.

```kira
// Trap: Bug if called with invalid index
let get: fn(List[i32], i32) -> i32 = fn(list: List[i32], index: i32) -> i32 {
    assert(index >= 0, "Index must be non-negative")
    // ... implementation
}

// Error: File may legitimately not exist
effect fn read_file(path: string) -> IO[Result[string, IoError]] {
    // ... implementation
}
```

---

## 8. API Design

### 8.1 Function Design

**A1**: Functions SHOULD have at most 4 parameters. Use config records for more:

```kira
// BAD: Too many parameters
let create_window: fn(string, i32, i32, i32, i32, bool) -> Window = ...

// GOOD: Config record
type WindowConfig = {
    title: string,
    x: i32,
    y: i32,
    width: i32,
    height: i32,
    fullscreen: bool
}

let create_window: fn(WindowConfig) -> Window = ...
```

### 8.2 Return Types

**A2**: Return owned values. Pure functions always return new values.

```kira
// Transform and return new list
let map[A, B]: fn(List[A], fn(A) -> B) -> List[B] = ...

// Return new record with modifications
let with_name: fn(User, string) -> User = fn(user: User, name: string) -> User {
    return User {
        id: user.id,
        name: name,
        email: user.email,
        created_at: user.created_at
    }
}
```

### 8.3 Generic Functions

**A3**: Use type parameters for reusable functions:

```kira
// Generic identity
let identity[T]: fn(T) -> T = fn(x: T) -> T {
    return x
}

// Generic with constraint (if supported)
let max[T]: fn(T, T) -> T = fn(a: T, b: T) -> T
where T: Ord
{
    if a > b {
        return a
    }
    return b
}
```

### 8.4 Module Visibility

**A4**: Make items private by default. Only expose what's necessary:

```kira
module http_client

// Public API
pub type Client = { ... }
pub let create: fn() -> Client = ...
pub effect fn get: fn(Client, string) -> IO[Result[Response, HttpError]] = ...

// Private implementation
type ConnectionPool = { ... }
let manage_pool: fn(ConnectionPool) -> ConnectionPool = ...
```

### 8.5 Documentation

**A5**: Document all public items:

```kira
/// Parses an integer from a string.
///
/// # Arguments
/// * `s` - The string to parse
///
/// # Returns
/// The parsed integer, or ParseError if invalid.
///
/// # Example
/// ```
/// let result: Result[i32, ParseError] = parse_int("42")
/// ```
pub let parse_int: fn(string) -> Result[i32, ParseError] = ...
```

---

## 9. Security

### 9.1 Input Validation

**S1**: Validate ALL external input before use:
- User input (CLI, stdin, GUI)
- File contents
- Network data
- Environment variables

```kira
let process_input: fn(string) -> Result[Command, InputError] =
    fn(input: string) -> Result[Command, InputError] {
        // Validate length
        if std.string.length(input) > MAX_INPUT_LENGTH {
            return Err(InputError.TooLong)
        }
        // Validate format
        if not is_valid_format(input) {
            return Err(InputError.InvalidFormat)
        }
        return parse_command(input)
    }
```

### 9.2 Integer Safety

**S2**: Use appropriate overflow handling:

```kira
// Use checked arithmetic for external input
let safe_add: fn(i32, i32) -> Option[i32] =
    fn(a: i32, b: i32) -> Option[i32] {
        return checked_add(a, b)
    }

// Use saturating arithmetic when overflow should clamp
let bounded_add: fn(i32, i32) -> i32 =
    fn(a: i32, b: i32) -> i32 {
        return saturating_add(a, b)
    }
```

### 9.3 Path Safety

**S3**: Validate file paths to prevent directory traversal:

```kira
let safe_path: fn(string, string) -> Result[string, PathError] =
    fn(base: string, user_path: string) -> Result[string, PathError] {
        if std.string.contains(user_path, "..") {
            return Err(PathError.TraversalAttempt)
        }
        let full: string = join_path(base, user_path)
        let canonical: string = canonicalize(full)?
        if not std.string.starts_with(canonical, base) {
            return Err(PathError.OutsideBase)
        }
        return Ok(canonical)
    }
```

### 9.4 Command Execution

**S4**: Never pass untrusted input to shell commands:

```kira
// BAD: Command injection
effect fn bad_exec(user_input: string) -> IO[void] {
    system("process " + user_input)  // DANGEROUS
    return
}

// GOOD: Use argument arrays
effect fn safe_exec(filename: string) -> IO[Result[void, ExecError]] {
    if not is_safe_filename(filename) {
        return Err(ExecError.InvalidFilename)
    }
    return exec("process", [filename])
}
```

### 9.5 Secrets

**S5**: Never log or expose:
- Passwords, tokens, API keys
- Private keys, certificates
- PII without explicit requirement

---

## 10. Code Organization

### 10.1 Module Structure

```
project/
├── src/
│   ├── main.ki           # Entry point (if executable)
│   ├── lib.ki            # Library root (if library)
│   ├── module_a.ki       # Top-level modules
│   ├── module_b.ki
│   └── submodule/        # Nested modules
│       ├── mod.ki        # Submodule root
│       └── helper.ki
├── tests/
│   ├── test_module_a.ki
│   └── test_module_b.ki
└── README.md
```

### 10.2 Import Organization

Order imports as follows:

1. Standard library imports
2. Third-party library imports
3. Project imports

```kira
// Standard library
import std.list.{ map, filter, fold }
import std.option.{ Option, Some, None }

// Third-party
import http.{ Client, Request }

// Project modules
import project.config.Config
import project.utils.helpers
```

### 10.3 File Length

**F1**: Keep files under 500 lines. Split large files into submodules.

**F2**: Keep functions under 50 lines. Extract helpers for complex logic.

### 10.4 Section Comments

For longer files, use section comments:

```kira
// ============================================================
// Types
// ============================================================

type User = { ... }
type UserError = ...

// ============================================================
// Public API
// ============================================================

pub let create_user: fn(...) = ...

// ============================================================
// Private Helpers
// ============================================================

let validate_name: fn(string) -> bool = ...
```

---

## 11. Documentation

### 11.1 Function Documentation

**D1**: Document all public functions:

```kira
/// Creates a new user with the given name.
///
/// # Arguments
/// * `name` - The user's display name (1-32 characters)
///
/// # Returns
/// A new User, or UserError if name is invalid.
///
/// # Example
/// ```
/// let user: Result[User, UserError] = create_user("Alice")
/// ```
pub let create_user: fn(string) -> Result[User, UserError] = ...
```

### 11.2 Required Documentation Elements

For public functions:
- Brief description (first line)
- Parameter descriptions (for non-obvious params)
- Return value description
- Error conditions (if returns Result)
- Example (for complex APIs)

### 11.3 Module Documentation

```kira
//! HTTP client library for making web requests.
//!
//! This module provides a simple interface for HTTP GET, POST, PUT, and DELETE
//! requests with support for timeouts, headers, and JSON bodies.
//!
//! # Example
//! ```
//! let client: Client = create_client()
//! let response: Result[Response, HttpError] = get(client, "https://api.example.com")
//! ```

module http_client
```

### 11.4 Comment Guidelines

**D2**: Explain WHY, not WHAT. The code shows what; comments explain intent.

```kira
// BAD: Restates the code
// Add one to counter
let new_counter: i32 = counter + 1

// GOOD: Explains why
// Track retry attempts for exponential backoff
let new_retry_count: i32 = retry_count + 1
```

**D3**: Keep comments up to date. Wrong comments are worse than no comments.

---

## 12. Testing

### 12.1 Test Organization

**T1**: One test file per module: `tests/test_<module>.ki`

**T2**: Test function naming: `test_<function>_<scenario>`

```kira
// tests/test_user.ki

let test_create_user_success: fn() -> void = fn() -> void {
    let user: Result[User, UserError] = create_user("Alice")
    assert(is_ok(user))
    assert(get_name(unwrap(user)) == "Alice")
    return
}

let test_create_user_empty_name: fn() -> void = fn() -> void {
    let result: Result[User, UserError] = create_user("")
    assert(is_err(result))
    return
}
```

### 12.2 Required Test Categories

Every module needs tests for:

1. **Happy path** - Normal successful operation
2. **Edge cases** - Empty input, zero values, maximum values, boundaries
3. **Error conditions** - Invalid input, missing resources
4. **Property tests** - Invariants that should always hold

### 12.3 Test Structure

Use Arrange-Act-Assert pattern:

```kira
let test_map_doubles_values: fn() -> void = fn() -> void {
    // Arrange
    let input: List[i32] = Cons(1, Cons(2, Cons(3, Nil)))

    // Act
    let result: List[i32] = map[i32, i32](input, fn(x: i32) -> i32 { return x * 2 })

    // Assert
    assert(list_equals(result, Cons(2, Cons(4, Cons(6, Nil)))))
    return
}
```

### 12.4 Property-Based Testing

**T3**: For pure functions, test properties that should always hold:

```kira
// Property: map preserves length
let test_map_preserves_length: fn() -> void = fn() -> void {
    for _ in range(0, 100) {
        let input: List[i32] = random_list()
        let output: List[i32] = map[i32, i32](input, fn(x: i32) -> i32 { return x * 2 })
        assert(length(input) == length(output))
    }
    return
}

// Property: filter result is subset of input
let test_filter_subset: fn() -> void = fn() -> void {
    for _ in range(0, 100) {
        let input: List[i32] = random_list()
        let output: List[i32] = filter[i32](input, fn(x: i32) -> bool { return x > 0 })
        assert(length(output) <= length(input))
    }
    return
}
```

### 12.5 Testing Pure Functions

**T4**: Pure functions are easy to test - no mocks needed:

```kira
// Pure function - easy to test
let test_factorial: fn() -> void = fn() -> void {
    assert(factorial(0) == 1)
    assert(factorial(1) == 1)
    assert(factorial(5) == 120)
    assert(factorial(10) == 3628800)
    return
}
```

### 12.6 Testing Effect Functions

**T5**: Effect functions need IO testing infrastructure:

```kira
effect fn test_read_file_not_found() -> IO[void] {
    let result: Result[string, IoError] = std.fs.read_file("/nonexistent")
    assert(is_err(result))
    return
}
```

---

## 13. Logging

### 13.1 Log Levels

| Level | Use For |
|-------|---------|
| `error` | Failures preventing operation |
| `warn` | Unexpected but recoverable |
| `info` | Significant events (startup, config) |
| `debug` | Troubleshooting details |
| `trace` | Detailed execution flow |

### 13.2 Logging is an Effect

**L1**: Logging performs IO, so it must be in effect functions:

```kira
// Logging is effectful
effect fn process_request(req: Request) -> IO[Response] {
    std.log.info("Processing request: id={req.id}")
    let response: Response = handle(req)
    std.log.info("Request complete: id={req.id}, status={response.status}")
    return response
}
```

### 13.3 Message Format

**L2**: Include context in messages:

```kira
// GOOD
std.log.error("Failed to load config: path={path}, error={err}")
std.log.info("Server started: port={port}")

// BAD
std.log.error("Load failed")
std.log.info("Started")
```

### 13.4 What NOT to Log

**L3**: Never log secrets (passwords, tokens, API keys)

**L4**: Never log PII without explicit requirement

**L5**: Never log in tight loops:

```kira
// BAD: Millions of log entries
for item in items {
    std.log.debug("Processing {item}")
}

// GOOD: Log summary
std.log.debug("Processing {length(items)} items")
```

---

## Quick Reference Checklist

Before committing code, verify:

- [ ] **Naming**: Types are `PascalCase`, functions are `snake_case`
- [ ] **Types**: All bindings have explicit type annotations
- [ ] **Returns**: All functions use explicit `return` statements
- [ ] **Purity**: Pure functions don't call effectful functions
- [ ] **Patterns**: All pattern matches are exhaustive
- [ ] **Errors**: All fallible operations return `Result[T, E]`
- [ ] **Validation**: All external input is validated before use
- [ ] **Documentation**: All public items are documented
- [ ] **Tests**: All public functions have tests
- [ ] **No secrets**: No passwords, tokens, or keys in code/logs

---

*Kira-Toolkit Standards v0.1.0 - Initial Release*
