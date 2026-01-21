---
globs: ["**/*.ki"]
---

# Security Rules

## Input Validation

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

## Type-Driven Validation

**S2**: Use types to represent validated data:

```kira
// Unvalidated input
type RawEmail = string

// Validated input (construction validates)
type ValidEmail = { value: string }

let parse_email: fn(string) -> Result[ValidEmail, EmailError] =
    fn(s: string) -> Result[ValidEmail, EmailError] {
        if not is_valid_email_format(s) {
            return Err(EmailError.InvalidFormat)
        }
        return Ok(ValidEmail { value: s })
    }

// Functions require validated type
let send_email: fn(ValidEmail, string) -> Result[void, SendError] = ...
```

## Integer Safety

**S3**: Use appropriate overflow handling:

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

## Path Safety

**S4**: Validate file paths to prevent directory traversal:

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

## Command Execution

**S5**: Never pass untrusted input to shell commands:

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

## Secrets

**S6**: Never log or expose:
- Passwords, tokens, API keys
- Private keys, certificates
- PII without explicit requirement

```kira
// BAD: Logging secrets
effect fn bad_auth(password: string) -> IO[void] {
    std.log.debug("Authenticating with password: {password}")  // NEVER!
    return
}

// GOOD: Mask sensitive data
effect fn good_auth(password: string) -> IO[void] {
    std.log.debug("Authenticating user")  // No secret in log
    return
}
```

## Whitelist Over Blacklist

**S7**: Prefer whitelists over blacklists:

```kira
// BAD: Blacklist (easy to miss dangerous input)
let is_safe_char_blacklist: fn(char) -> bool = fn(c: char) -> bool {
    return c != '<' and c != '>'  // Missing many dangerous chars!
}

// GOOD: Whitelist (explicitly allow only safe chars)
let is_safe_char_whitelist: fn(char) -> bool = fn(c: char) -> bool {
    return is_alphanumeric(c) or c == ' ' or c == '.'
}
```

## Security Checklist

- [ ] All external input validated
- [ ] Types enforce validation state
- [ ] Path traversal prevented
- [ ] No shell injection possible
- [ ] Secrets never logged
- [ ] Whitelist approach used
- [ ] Integer overflow handled
