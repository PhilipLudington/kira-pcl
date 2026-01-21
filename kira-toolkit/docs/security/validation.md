# Input Validation Patterns

Comprehensive patterns for validating external input in Kira.

## The Validation Principle

> Never trust external input. Validate everything at system boundaries.

External input includes:
- User input (CLI, GUI, stdin)
- File contents
- Network data
- Environment variables
- Database results (if from untrusted sources)

## Pattern 1: Validation at Boundaries

Validate input as early as possible:

```kira
/// Entry point for HTTP requests
effect fn handle_request(raw: RawRequest) -> IO[Response] {
    // Validate immediately
    match validate_request(raw) {
        Ok(req) => { return process_request(req) }
        Err(e) => { return Response.bad_request(error_message(e)) }
    }
}

// Pure validation function
let validate_request: fn(RawRequest) -> Result[ValidRequest, ValidationError] =
    fn(raw: RawRequest) -> Result[ValidRequest, ValidationError] {
        let method: HttpMethod = validate_method(raw.method)?
        let path: ValidPath = validate_path(raw.path)?
        let headers: ValidHeaders = validate_headers(raw.headers)?
        return Ok(ValidRequest { method: method, path: path, headers: headers })
    }
```

## Pattern 2: Type-Driven Validation

Use types to represent validation state:

```kira
// Unvalidated - raw user input
type RawInput = { value: string }

// Validated - guaranteed safe
type ValidatedInput = { value: string }

// Only way to create ValidatedInput is through validation
let validate_input: fn(RawInput) -> Result[ValidatedInput, ValidationError] =
    fn(raw: RawInput) -> Result[ValidatedInput, ValidationError] {
        let v: string = raw.value
        if std.string.length(v) > MAX_LENGTH {
            return Err(ValidationError.TooLong(MAX_LENGTH))
        }
        if not is_safe_characters(v) {
            return Err(ValidationError.UnsafeCharacters)
        }
        return Ok(ValidatedInput { value: v })
    }

// Functions require validated type - cannot be called with raw input
let process: fn(ValidatedInput) -> Output = ...
```

## Pattern 3: String Validators

Build reusable string validators:

```kira
type StringConstraints = {
    min_length: Option[i32],
    max_length: Option[i32],
    allowed_chars: Option[string],
    pattern: Option[Regex]
}

let validate_string: fn(string, StringConstraints) -> Result[string, ValidationError] =
    fn(s: string, constraints: StringConstraints) -> Result[string, ValidationError] {
        let len: i32 = std.string.length(s)

        match constraints.min_length {
            Some(min) if len < min => {
                return Err(ValidationError.TooShort { min: min, actual: len })
            }
            _ => {}
        }

        match constraints.max_length {
            Some(max) if len > max => {
                return Err(ValidationError.TooLong { max: max, actual: len })
            }
            _ => {}
        }

        match constraints.allowed_chars {
            Some(allowed) => {
                for c in std.string.chars(s) {
                    if not std.string.contains(allowed, c) {
                        return Err(ValidationError.InvalidChar(c))
                    }
                }
            }
            None => {}
        }

        match constraints.pattern {
            Some(regex) => {
                if not regex_match(regex, s) {
                    return Err(ValidationError.PatternMismatch)
                }
            }
            None => {}
        }

        return Ok(s)
    }

// Predefined validators
let USERNAME_CONSTRAINTS: StringConstraints = StringConstraints {
    min_length: Some(3),
    max_length: Some(32),
    allowed_chars: Some("abcdefghijklmnopqrstuvwxyz0123456789_"),
    pattern: None
}

let validate_username: fn(string) -> Result[string, ValidationError] =
    fn(s: string) -> Result[string, ValidationError] {
        return validate_string(s, USERNAME_CONSTRAINTS)
    }
```

## Pattern 4: Numeric Validators

```kira
type NumericConstraints = {
    min: Option[i32],
    max: Option[i32],
    allow_negative: bool
}

let validate_int: fn(i32, NumericConstraints) -> Result[i32, ValidationError] =
    fn(n: i32, constraints: NumericConstraints) -> Result[i32, ValidationError] {
        if not constraints.allow_negative and n < 0 {
            return Err(ValidationError.NegativeNotAllowed)
        }

        match constraints.min {
            Some(min) if n < min => {
                return Err(ValidationError.BelowMinimum { min: min, actual: n })
            }
            _ => {}
        }

        match constraints.max {
            Some(max) if n > max => {
                return Err(ValidationError.AboveMaximum { max: max, actual: n })
            }
            _ => {}
        }

        return Ok(n)
    }

// Usage
let AGE_CONSTRAINTS: NumericConstraints = NumericConstraints {
    min: Some(0),
    max: Some(150),
    allow_negative: false
}

let validate_age: fn(i32) -> Result[i32, ValidationError] =
    fn(age: i32) -> Result[i32, ValidationError] {
        return validate_int(age, AGE_CONSTRAINTS)
    }
```

## Pattern 5: Email Validation

```kira
type Email = { local: string, domain: string }

let parse_email: fn(string) -> Result[Email, EmailError] =
    fn(s: string) -> Result[Email, EmailError] {
        // Length check
        if std.string.length(s) > 254 {
            return Err(EmailError.TooLong)
        }

        // Find @ symbol
        match std.string.find(s, "@") {
            None => { return Err(EmailError.MissingAt) }
            Some(at_pos) => {
                let local: string = std.string.substring(s, 0, at_pos)
                let domain: string = std.string.substring(s, at_pos + 1, std.string.length(s))

                // Validate local part
                if std.string.length(local) == 0 or std.string.length(local) > 64 {
                    return Err(EmailError.InvalidLocalPart)
                }

                // Validate domain
                if not is_valid_domain(domain) {
                    return Err(EmailError.InvalidDomain)
                }

                return Ok(Email { local: local, domain: domain })
            }
        }
    }

let is_valid_domain: fn(string) -> bool = fn(domain: string) -> bool {
    if std.string.length(domain) == 0 {
        return false
    }
    if not std.string.contains(domain, ".") {
        return false
    }
    // Additional validation...
    return true
}
```

## Pattern 6: Structured Data Validation

Validate complex data structures:

```kira
type RawRegistration = {
    username: string,
    email: string,
    password: string,
    age: i32
}

type ValidRegistration = {
    username: ValidUsername,
    email: Email,
    password: ValidPassword,
    age: i32
}

let validate_registration: fn(RawRegistration) -> Result[ValidRegistration, List[FieldError]] =
    fn(raw: RawRegistration) -> Result[ValidRegistration, List[FieldError]] {
        var errors: List[FieldError] = Nil

        let username_result: Result[ValidUsername, ValidationError] = validate_username(raw.username)
        let email_result: Result[Email, EmailError] = parse_email(raw.email)
        let password_result: Result[ValidPassword, ValidationError] = validate_password(raw.password)
        let age_result: Result[i32, ValidationError] = validate_age(raw.age)

        // Collect errors
        match username_result {
            Err(e) => { errors = Cons(FieldError.new("username", e), errors) }
            Ok(_) => {}
        }
        match email_result {
            Err(e) => { errors = Cons(FieldError.new("email", e), errors) }
            Ok(_) => {}
        }
        match password_result {
            Err(e) => { errors = Cons(FieldError.new("password", e), errors) }
            Ok(_) => {}
        }
        match age_result {
            Err(e) => { errors = Cons(FieldError.new("age", e), errors) }
            Ok(_) => {}
        }

        match errors {
            Cons(_, _) => { return Err(errors) }
            Nil => {
                return Ok(ValidRegistration {
                    username: unwrap(username_result),
                    email: unwrap(email_result),
                    password: unwrap(password_result),
                    age: unwrap(age_result)
                })
            }
        }
    }
```

## Pattern 7: Sanitization

Sometimes sanitize rather than reject:

```kira
/// Sanitizes a string for safe display in HTML
let sanitize_html: fn(string) -> string = fn(input: string) -> string {
    let s1: string = std.string.replace(input, "&", "&amp;")
    let s2: string = std.string.replace(s1, "<", "&lt;")
    let s3: string = std.string.replace(s2, ">", "&gt;")
    let s4: string = std.string.replace(s3, "\"", "&quot;")
    let s5: string = std.string.replace(s4, "'", "&#x27;")
    return s5
}

/// Removes non-alphanumeric characters from filename
let sanitize_filename: fn(string) -> string = fn(input: string) -> string {
    let filtered: List[char] = filter[char](
        std.string.chars(input),
        fn(c: char) -> bool {
            return is_alphanumeric(c) or c == '.' or c == '-' or c == '_'
        }
    )
    return std.string.from_chars(filtered)
}

/// Truncates string to maximum length
let truncate: fn(string, i32) -> string = fn(input: string, max_len: i32) -> string {
    if std.string.length(input) <= max_len {
        return input
    }
    return std.string.substring(input, 0, max_len)
}
```

## Pattern 8: Whitelist Validation

Prefer whitelists over blacklists:

```kira
// BAD: Blacklist (easy to miss dangerous input)
let is_safe_blacklist: fn(char) -> bool = fn(c: char) -> bool {
    return c != '<' and c != '>' and c != '&'  // Missing many dangerous chars!
}

// GOOD: Whitelist (explicitly allow only safe chars)
let is_safe_whitelist: fn(char) -> bool = fn(c: char) -> bool {
    return is_alphanumeric(c) or c == ' ' or c == '.' or c == ','
}

// GOOD: Enum whitelist for operations
type AllowedOperation =
    | Read
    | Write
    | Delete

let perform_operation: fn(AllowedOperation, Resource) -> Result[void, Error] =
    fn(op: AllowedOperation, resource: Resource) -> Result[void, Error] {
        // Only whitelisted operations can be performed
        match op {
            Read => { return read_resource(resource) }
            Write => { return write_resource(resource) }
            Delete => { return delete_resource(resource) }
        }
    }
```

## Validation Error Messages

Provide helpful but secure messages:

```kira
type ValidationError =
    | TooShort { min: i32, actual: i32 }
    | TooLong { max: i32, actual: i32 }
    | InvalidFormat
    | InvalidChar(char)

// User-facing message (safe to display)
let user_message: fn(ValidationError) -> string = fn(err: ValidationError) -> string {
    var result: string
    match err {
        TooShort { min, .. } => { result = "Must be at least {min} characters" }
        TooLong { max, .. } => { result = "Must be at most {max} characters" }
        InvalidFormat => { result = "Invalid format" }
        InvalidChar(_) => { result = "Contains invalid characters" }  // Don't reveal which char
    }
    return result
}

// Internal message (for logging)
let internal_message: fn(ValidationError) -> string = fn(err: ValidationError) -> string {
    var result: string
    match err {
        TooShort { min, actual } => { result = "Too short: min={min}, actual={actual}" }
        TooLong { max, actual } => { result = "Too long: max={max}, actual={actual}" }
        InvalidFormat => { result = "Invalid format" }
        InvalidChar(c) => { result = "Invalid char: {c}" }
    }
    return result
}
```

## Validation Checklist

- [ ] All external input validated at boundaries
- [ ] Types enforce validation state
- [ ] Length limits enforced
- [ ] Character restrictions applied (whitelist preferred)
- [ ] Format validation for structured data
- [ ] Range validation for numeric values
- [ ] Error messages don't leak sensitive info
- [ ] Sanitization used where rejection isn't appropriate
