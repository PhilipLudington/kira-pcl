# Error Handling Patterns

Patterns for robust error handling in Kira.

## The Error Model

Kira uses:
- **Result[T, E]**: For operations that can fail
- **Option[T]**: For values that may be absent
- **Pattern matching**: For handling different cases

## Pattern 1: Result for Fallible Operations

Any operation that can fail should return `Result[T, E]`:

```kira
type FileError =
    | NotFound(string)
    | PermissionDenied(string)
    | IoError(string)

effect fn read_file(path: string) -> IO[Result[string, FileError]] {
    if not file_exists(path) {
        return Err(FileError.NotFound(path))
    }
    if not is_readable(path) {
        return Err(FileError.PermissionDenied(path))
    }
    // ... read and return Ok(contents)
}
```

## Pattern 2: Error Propagation with ?

Use `?` to propagate errors up the call stack:

```kira
effect fn load_config(path: string) -> IO[Result[Config, ConfigError]] {
    let contents: string = read_file(path)?        // Propagates FileError
    let parsed: Toml = parse_toml(contents)?       // Propagates ParseError
    let validated: Config = validate(parsed)?      // Propagates ValidationError
    return Ok(validated)
}
```

The `?` operator:
1. If `Ok(value)`, extracts the value
2. If `Err(e)`, returns early with the error

## Pattern 3: Custom Error Types

Define meaningful error types for each module:

```kira
/// Errors that can occur during configuration.
type ConfigError =
    | NotFound(string)
    | ParseError { line: i32, column: i32, message: string }
    | ValidationError { field: string, message: string }
    | IoError(IoError)

let error_message: fn(ConfigError) -> string = fn(err: ConfigError) -> string {
    var result: string
    match err {
        NotFound(path) => { result = "Config not found: {path}" }
        ParseError { line, column, message } => {
            result = "Parse error at {line}:{column}: {message}"
        }
        ValidationError { field, message } => {
            result = "Invalid {field}: {message}"
        }
        IoError(e) => { result = "I/O error: {e}" }
    }
    return result
}
```

## Pattern 4: Option for Absent Values

Use `Option[T]` when a value may be absent (not an error):

```kira
let find_user: fn(List[User], i64) -> Option[User] =
    fn(users: List[User], id: i64) -> Option[User] {
        match users {
            Nil => { return None }
            Cons(user, tail) => {
                if user.id == id {
                    return Some(user)
                }
                return find_user(tail, id)
            }
        }
    }

let get_env: fn(string) -> Option[string] =
    fn(key: string) -> Option[string] {
        // Returns None if not set
    }
```

**Option vs Result:**
- `Option`: Value might not exist (not a failure)
- `Result`: Operation might fail (is a failure)

## Pattern 5: Pattern Matching Errors

Match on errors for specific handling:

```kira
let handle_config_error: fn(Result[Config, ConfigError]) -> Config =
    fn(result: Result[Config, ConfigError]) -> Config {
        var output: Config
        match result {
            Ok(config) => { output = config }
            Err(ConfigError.NotFound(_)) => {
                // Use defaults if config doesn't exist
                output = default_config()
            }
            Err(ConfigError.ParseError { line, message, .. }) => {
                // Log and use defaults
                log_parse_error(line, message)
                output = default_config()
            }
            Err(e) => {
                // For other errors, panic
                panic("Config error: {error_message(e)}")
            }
        }
        return output
    }
```

## Pattern 6: Fallback Chain

Try multiple approaches, falling back on error:

```kira
effect fn load_config() -> IO[Result[Config, ConfigError]] {
    // Try user config first
    let user_result: Result[Config, ConfigError] = load_config_file("~/.config/app.toml")
    match user_result {
        Ok(config) => { return Ok(config) }
        Err(_) => {
            // Fall back to system config
            let system_result: Result[Config, ConfigError] = load_config_file("/etc/app.toml")
            match system_result {
                Ok(config) => { return Ok(config) }
                Err(_) => {
                    // Fall back to defaults
                    return Ok(default_config())
                }
            }
        }
    }
}
```

## Pattern 7: Collecting Results

Process a collection, handling errors:

```kira
// Stop on first error
let process_all: fn(List[Item]) -> Result[void, ProcessError] =
    fn(items: List[Item]) -> Result[void, ProcessError] {
        match items {
            Nil => { return Ok(()) }
            Cons(item, tail) => {
                process(item)?
                return process_all(tail)
            }
        }
    }

// Collect all errors
let validate_all: fn(List[Item]) -> List[ValidationError] =
    fn(items: List[Item]) -> List[ValidationError] {
        match items {
            Nil => { return Nil }
            Cons(item, tail) => {
                let rest: List[ValidationError] = validate_all(tail)
                match validate(item) {
                    Ok(_) => { return rest }
                    Err(e) => { return Cons(e, rest) }
                }
            }
        }
    }
```

## Pattern 8: Context Enrichment

Add context when propagating errors:

```kira
type LoadError =
    | UserLoadFailed { user_id: i64, cause: string }

let load_user_data: fn(i64) -> Result[UserData, LoadError] =
    fn(user_id: i64) -> Result[UserData, LoadError] {
        let config_path: string = get_user_config_path(user_id)
        match load_config(config_path) {
            Ok(config) => { return Ok(parse_user_data(config)) }
            Err(e) => {
                return Err(LoadError.UserLoadFailed {
                    user_id: user_id,
                    cause: error_message(e)
                })
            }
        }
    }
```

## Pattern 9: Default Values

Provide defaults for optional values:

```kira
// Using ?? for Option
let port: i32 = get_env_int("PORT") ?? 8080

// Using ?? for Result
let config: Config = load_config() ?? default_config()

// Explicit default with match
let value: i32 = match parse_int(input) {
    Ok(n) => n
    Err(_) => 0
}
```

## Pattern 10: Assertions for Invariants

Use assert for conditions that indicate bugs:

```kira
let get_element: fn(List[i32], i32) -> i32 = fn(list: List[i32], index: i32) -> i32 {
    // Caller error if index is invalid
    assert(index >= 0, "Index must be non-negative")
    assert(index < length(list), "Index out of bounds")
    // ... return element
}

let divide: fn(i32, i32) -> i32 = fn(a: i32, b: i32) -> i32 {
    // Caller error if dividing by zero
    assert(b != 0, "Division by zero")
    return a / b
}
```

## Anti-Patterns to Avoid

### 1. Swallowing Errors

```kira
// BAD: Error silently ignored
let _ = save_data(data)

// GOOD: Handle or propagate
save_data(data)?
// or
match save_data(data) {
    Ok(_) => {}
    Err(e) => { log_error(e) }
}
```

### 2. Using Panic for Expected Failures

```kira
// BAD: Panic on expected condition
let parse: fn(string) -> i32 = fn(input: string) -> i32 {
    if not is_numeric(input) {
        panic("Invalid input")  // Wrong! This is expected
    }
    // ...
}

// GOOD: Return Result
let parse: fn(string) -> Result[i32, ParseError] = fn(input: string) -> Result[i32, ParseError] {
    if not is_numeric(input) {
        return Err(ParseError.InvalidFormat)
    }
    // ...
}
```

### 3. Stringly-Typed Errors

```kira
// BAD: Error as string
let load: fn() -> Result[Data, string] = fn() -> Result[Data, string] {
    return Err("Failed to load")  // No structure!
}

// GOOD: Typed error
let load: fn() -> Result[Data, LoadError] = fn() -> Result[Data, LoadError] {
    return Err(LoadError.NotFound { path: path })
}
```

## Error Handling Checklist

- [ ] Fallible operations return Result
- [ ] Option used for optional values
- [ ] Errors are propagated with ?
- [ ] Custom error types include context
- [ ] Errors are never silently swallowed
- [ ] Assert used only for invariants (bugs)
- [ ] Error messages are helpful but don't leak secrets
