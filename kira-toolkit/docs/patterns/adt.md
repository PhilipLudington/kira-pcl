# Algebraic Data Type Patterns

Patterns for designing effective algebraic data types in Kira.

## The ADT Model

Kira supports two kinds of algebraic data types:
- **Sum types**: "One of" several variants (tagged unions)
- **Product types**: "All of" several fields (records)

## Pattern 1: Sum Types for States

Model state machines with sum types:

```kira
type ConnectionState =
    | Disconnected
    | Connecting { address: string }
    | Connected { socket: Socket, address: string }
    | Disconnecting { reason: string }

let handle_state: fn(ConnectionState) -> string = fn(state: ConnectionState) -> string {
    var result: string
    match state {
        Disconnected => { result = "Not connected" }
        Connecting { address: addr } => { result = "Connecting to {addr}..." }
        Connected { socket: _, address: addr } => { result = "Connected to {addr}" }
        Disconnecting { reason: r } => { result = "Disconnecting: {r}" }
    }
    return result
}
```

## Pattern 2: Make Illegal States Unrepresentable

Design types so invalid combinations cannot exist:

```kira
// BAD: Allows invalid states
type BadUser = {
    email: Option[string],
    email_verified: bool  // Can be true when email is None!
}

// GOOD: Invalid states impossible
type EmailStatus =
    | NoEmail
    | Unverified(string)
    | Verified(string)

type GoodUser = {
    name: string,
    email: EmailStatus
}

// Only verified emails can be marked verified
let get_verified_email: fn(GoodUser) -> Option[string] =
    fn(user: GoodUser) -> Option[string] {
        var result: Option[string]
        match user.email {
            Verified(email) => { result = Some(email) }
            _ => { result = None }
        }
        return result
    }
```

## Pattern 3: Event Modeling

Model domain events with specific sum types:

```kira
// BAD: Generic event with stringly-typed data
type BadEvent = {
    event_type: string,
    data: string
}

// GOOD: Specific, type-safe events
type UserEvent =
    | UserCreated { user_id: i64, name: string, email: string }
    | UserUpdated { user_id: i64, changes: UserChanges }
    | UserDeleted { user_id: i64, reason: string }
    | UserLoggedIn { user_id: i64, ip_address: string }
    | UserLoggedOut { user_id: i64 }

let handle_event: fn(UserEvent) -> Result[void, Error] =
    fn(event: UserEvent) -> Result[void, Error] {
        var result: Result[void, Error]
        match event {
            UserCreated { user_id, name, email } => {
                result = create_user_record(user_id, name, email)
            }
            UserUpdated { user_id, changes } => {
                result = apply_changes(user_id, changes)
            }
            UserDeleted { user_id, reason } => {
                result = delete_user_record(user_id, reason)
            }
            UserLoggedIn { user_id, ip_address } => {
                result = record_login(user_id, ip_address)
            }
            UserLoggedOut { user_id } => {
                result = record_logout(user_id)
            }
        }
        return result
    }
```

## Pattern 4: Error Types

Design error types with context:

```kira
type DatabaseError =
    | ConnectionFailed { host: string, port: i32 }
    | QueryFailed { query: string, reason: string }
    | Timeout { operation: string, duration_ms: i64 }
    | NotFound { table: string, key: string }

type ApiError =
    | BadRequest { message: string }
    | Unauthorized { reason: string }
    | NotFound { resource: string }
    | InternalError { cause: DatabaseError }

let format_api_error: fn(ApiError) -> string = fn(err: ApiError) -> string {
    var result: string
    match err {
        BadRequest { message } => { result = "Bad request: {message}" }
        Unauthorized { reason } => { result = "Unauthorized: {reason}" }
        NotFound { resource } => { result = "Not found: {resource}" }
        InternalError { cause } => { result = "Internal error" }  // Don't expose internal details
    }
    return result
}
```

## Pattern 5: Option and Result Wrappers

Create domain-specific wrappers around Option and Result:

```kira
// Domain-specific result types
type ParseResult[T] = Result[T, ParseError]
type DbResult[T] = Result[T, DatabaseError]
type ApiResult[T] = Result[T, ApiError]

// Usage makes intent clear
let parse_user: fn(string) -> ParseResult[User] = ...
let fetch_user: fn(i64) -> DbResult[User] = ...
let get_user: fn(i64) -> ApiResult[User] = ...
```

## Pattern 6: Recursive Types

Model recursive structures:

```kira
// Linked list
type List[T] =
    | Cons(T, List[T])
    | Nil

// Binary tree
type Tree[T] =
    | Node { value: T, left: Tree[T], right: Tree[T] }
    | Leaf

// JSON-like structure
type Json =
    | JsonNull
    | JsonBool(bool)
    | JsonNumber(f64)
    | JsonString(string)
    | JsonArray(List[Json])
    | JsonObject(List[(string, Json)])
```

## Pattern 7: Smart Constructors

Control construction with validation:

```kira
// Internal representation
type Email = { value: string }

// Smart constructor validates on creation
let create_email: fn(string) -> Result[Email, ValidationError] =
    fn(s: string) -> Result[Email, ValidationError] {
        if not std.string.contains(s, "@") {
            return Err(ValidationError.InvalidFormat("Missing @"))
        }
        if std.string.length(s) > 254 {
            return Err(ValidationError.TooLong)
        }
        return Ok(Email { value: s })
    }

// Usage ensures all Emails are valid
let send_email: fn(Email, string) -> Result[void, SendError] = ...
```

## Pattern 8: Builder Pattern with Records

Build complex records incrementally:

```kira
type HttpRequest = {
    method: string,
    url: string,
    headers: List[(string, string)],
    body: Option[string]
}

type RequestBuilder = {
    method: Option[string],
    url: Option[string],
    headers: List[(string, string)],
    body: Option[string]
}

let new_builder: fn() -> RequestBuilder = fn() -> RequestBuilder {
    return RequestBuilder {
        method: None,
        url: None,
        headers: Nil,
        body: None
    }
}

let with_method: fn(RequestBuilder, string) -> RequestBuilder =
    fn(b: RequestBuilder, m: string) -> RequestBuilder {
        return RequestBuilder {
            method: Some(m),
            url: b.url,
            headers: b.headers,
            body: b.body
        }
    }

let with_url: fn(RequestBuilder, string) -> RequestBuilder =
    fn(b: RequestBuilder, u: string) -> RequestBuilder {
        return RequestBuilder {
            method: b.method,
            url: Some(u),
            headers: b.headers,
            body: b.body
        }
    }

let build: fn(RequestBuilder) -> Result[HttpRequest, BuildError] =
    fn(b: RequestBuilder) -> Result[HttpRequest, BuildError] {
        match (b.method, b.url) {
            (Some(m), Some(u)) => {
                return Ok(HttpRequest {
                    method: m,
                    url: u,
                    headers: b.headers,
                    body: b.body
                })
            }
            (None, _) => { return Err(BuildError.MissingMethod) }
            (_, None) => { return Err(BuildError.MissingUrl) }
        }
    }
```

## Pattern 9: Phantom Types

Use type parameters for compile-time safety (if supported):

```kira
// Marker types
type Unvalidated = {}
type Validated = {}

// Input with validation state in type
type Input[State] = { value: string }

let validate: fn(Input[Unvalidated]) -> Result[Input[Validated], ValidationError] =
    fn(input: Input[Unvalidated]) -> Result[Input[Validated], ValidationError] {
        if is_valid(input.value) {
            return Ok(Input { value: input.value })
        }
        return Err(ValidationError.Invalid)
    }

// Only accepts validated input
let process: fn(Input[Validated]) -> Output = ...
```

## Pattern 10: Type Aliases for Clarity

Create meaningful aliases:

```kira
// Domain types
type UserId = i64
type PostId = i64
type Timestamp = i64

// Collection aliases
type UserList = List[User]
type PostMap = Map[PostId, Post]

// Result aliases
type UserResult = Result[User, UserError]
```

## ADT Design Checklist

- [ ] Sum types used for "one of" relationships
- [ ] Product types used for "all of" relationships
- [ ] Invalid states are unrepresentable
- [ ] Types are specific, not overly general
- [ ] Error types include context
- [ ] Smart constructors validate data
- [ ] Type aliases improve readability
