---
globs: ["**/*.ki"]
---

# API Design Rules

## Function Parameters

**A1**: Maximum 4 parameters. Use config records for more:

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

## Return Types

**A2**: Return owned values. Pure functions always return new values:

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

## Generic Functions

**A3**: Use type parameters for reusable functions:

```kira
// Generic identity
let identity[T]: fn(T) -> T = fn(x: T) -> T {
    return x
}

// Generic pair creation
let make_pair[A, B]: fn(A, B) -> Pair[A, B] = fn(a: A, b: B) -> Pair[A, B] {
    return Pair { first: a, second: b }
}
```

## Module Visibility

**A4**: Private by default. Only expose what's necessary:

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

## Documentation

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

## Pure vs Effect APIs

**A6**: Separate pure logic from effectful operations:

```kira
// Pure: data transformation
let process_data: fn(RawData) -> ProcessedData = ...

// Effect: IO operations
effect fn fetch_data: fn(Url) -> IO[Result[RawData, Error]] = ...

// Combine in effect context
effect fn fetch_and_process: fn(Url) -> IO[Result[ProcessedData, Error]] =
    fn(url: Url) -> IO[Result[ProcessedData, Error]] {
        let raw: RawData = fetch_data(url)?
        return Ok(process_data(raw))
    }
```

## Higher-Order Function APIs

**A7**: Accept functions as parameters for flexibility:

```kira
// Accept transformation function
let map[A, B]: fn(List[A], fn(A) -> B) -> List[B] = ...

// Accept predicate function
let filter[A]: fn(List[A], fn(A) -> bool) -> List[A] = ...

// Accept combining function
let fold[A, B]: fn(List[A], B, fn(B, A) -> B) -> B = ...
```

## API Design Checklist

- [ ] Functions have at most 4 parameters
- [ ] Config records used for many options
- [ ] Pure logic separated from effects
- [ ] Public items are documented
- [ ] Generic functions used for reusability
- [ ] Private by default, public only when needed
