---
globs: ["**/*.ki"]
---

# Error Handling Rules

## Result Type

All fallible operations MUST return `Result[T, E]`:

```kira
let parse_int: fn(string) -> Result[i32, ParseError] =
    fn(s: string) -> Result[i32, ParseError] {
        // ... parsing logic
    }

effect fn read_file(path: string) -> IO[Result[string, IoError]] {
    return std.fs.read_file(path)
}
```

## Option Type

Use `Option[T]` for values that may be absent (not errors):

```kira
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

**E1**: Use `Option` when absence is normal, `Result` when absence is an error.

## Error Propagation

Use `?` to propagate errors up the call stack:

```kira
effect fn load_config(path: string) -> IO[Result[Config, ConfigError]] {
    let content: string = std.fs.read_file(path)?    // Propagates IoError
    let parsed: Toml = parse_toml(content)?          // Propagates ParseError
    return Ok(validate(parsed))
}
```

**E2**: Use `?` for error propagation instead of explicit matching when appropriate.

## Custom Error Types

Define meaningful error types for each module:

```kira
type ConfigError =
    | NotFound(string)
    | ParseError { line: i32, message: string }
    | ValidationError(string)
    | IoError(IoError)
```

**E3**: Include context in error variants (file path, line number, etc.).

## Error Handling Patterns

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

## Traps vs Errors

- **Traps**: Bugs that should never happen - use `assert()`, `panic()`
- **Errors**: Expected failures - use `Result[T, E]`

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

## Error Handling Checklist

- [ ] Fallible functions return `Result[T, E]`
- [ ] `?` used for error propagation
- [ ] Custom error types include context
- [ ] Errors are not silently swallowed
- [ ] Option used for absent values, not errors
- [ ] Traps used only for programming errors
