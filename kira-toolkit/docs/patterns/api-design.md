# API Design Patterns

Patterns for designing clean, usable APIs in Kira.

## The Functional API Model

Kira APIs should:
- Be pure where possible
- Use explicit types
- Separate pure and effectful operations
- Leverage algebraic data types

## Pattern 1: Pure Core, Effectful Shell

Separate pure logic from IO:

```kira
// Pure core - business logic
let calculate_discount: fn(Order) -> f64 = fn(order: Order) -> f64 {
    let total: f64 = sum_items(order.items)
    if total > 100.0 {
        return total * 0.1
    }
    return 0.0
}

let apply_discount: fn(Order, f64) -> Order = fn(order: Order, discount: f64) -> Order {
    return Order {
        items: order.items,
        total: order.total - discount,
        discount_applied: discount
    }
}

// Effectful shell - IO operations
effect fn process_order(order_id: i64) -> IO[Result[Order, OrderError]] {
    let order: Order = fetch_order(order_id)?
    let discount: f64 = calculate_discount(order)
    let discounted: Order = apply_discount(order, discount)
    save_order(discounted)?
    return Ok(discounted)
}
```

## Pattern 2: Config Records for Complex Parameters

Use records instead of many parameters:

```kira
// BAD: Too many parameters
let create_connection: fn(string, i32, string, string, bool, i32) -> Connection = ...

// GOOD: Config record
type ConnectionConfig = {
    host: string,
    port: i32,
    username: string,
    password: string,
    use_ssl: bool,
    timeout_ms: i32
}

let default_config: fn() -> ConnectionConfig = fn() -> ConnectionConfig {
    return ConnectionConfig {
        host: "localhost",
        port: 5432,
        username: "",
        password: "",
        use_ssl: true,
        timeout_ms: 30000
    }
}

let create_connection: fn(ConnectionConfig) -> Result[Connection, ConnectionError] = ...

// Usage with modification
let config: ConnectionConfig = ConnectionConfig {
    host: "db.example.com",
    port: 5432,
    username: "app",
    password: env_password,
    use_ssl: true,
    timeout_ms: 60000
}
let conn: Result[Connection, ConnectionError] = create_connection(config)
```

## Pattern 3: Smart Constructors

Control type construction with validation:

```kira
// Private internal type
type PositiveInt = { value: i32 }

// Public smart constructor
let positive_int: fn(i32) -> Result[PositiveInt, ValidationError] =
    fn(n: i32) -> Result[PositiveInt, ValidationError] {
        if n <= 0 {
            return Err(ValidationError.NotPositive(n))
        }
        return Ok(PositiveInt { value: n })
    }

// Accessor
let get_value: fn(PositiveInt) -> i32 = fn(p: PositiveInt) -> i32 {
    return p.value
}

// Functions that need positive integers use the type
let divide_by: fn(i32, PositiveInt) -> i32 = fn(n: i32, divisor: PositiveInt) -> i32 {
    return n / divisor.value  // Safe - always positive
}
```

## Pattern 4: Builder Pattern

Build complex objects incrementally:

```kira
type QueryBuilder = {
    table: Option[string],
    columns: List[string],
    where_clause: Option[string],
    limit: Option[i32]
}

let new_query: fn() -> QueryBuilder = fn() -> QueryBuilder {
    return QueryBuilder {
        table: None,
        columns: Nil,
        where_clause: None,
        limit: None
    }
}

let from: fn(QueryBuilder, string) -> QueryBuilder =
    fn(b: QueryBuilder, table: string) -> QueryBuilder {
        return QueryBuilder {
            table: Some(table),
            columns: b.columns,
            where_clause: b.where_clause,
            limit: b.limit
        }
    }

let select: fn(QueryBuilder, List[string]) -> QueryBuilder =
    fn(b: QueryBuilder, cols: List[string]) -> QueryBuilder {
        return QueryBuilder {
            table: b.table,
            columns: cols,
            where_clause: b.where_clause,
            limit: b.limit
        }
    }

let build: fn(QueryBuilder) -> Result[Query, BuildError] =
    fn(b: QueryBuilder) -> Result[Query, BuildError] {
        match b.table {
            None => { return Err(BuildError.MissingTable) }
            Some(t) => {
                return Ok(Query {
                    sql: build_sql(t, b.columns, b.where_clause, b.limit)
                })
            }
        }
    }

// Usage
let query: Result[Query, BuildError] = build(
    select(from(new_query(), "users"), ["id", "name"])
)
```

## Pattern 5: Type-Safe Identifiers

Create distinct types for different IDs:

```kira
type UserId = { value: i64 }
type PostId = { value: i64 }
type CommentId = { value: i64 }

// These are distinct types - can't mix them up
let get_user: fn(UserId) -> Result[User, Error] = ...
let get_post: fn(PostId) -> Result[Post, Error] = ...
let get_comment: fn(CommentId) -> Result[Comment, Error] = ...

// Compile error: get_post(user_id) - types don't match
```

## Pattern 6: Result-Returning APIs

Always return Result for fallible operations:

```kira
// BAD: Returns null/None for errors
let parse_int_bad: fn(string) -> Option[i32] = ...  // Why did it fail?

// GOOD: Returns Result with error info
type ParseError =
    | EmptyString
    | InvalidCharacter { position: i32, char: char }
    | Overflow

let parse_int: fn(string) -> Result[i32, ParseError] = fn(s: string) -> Result[i32, ParseError] {
    if std.string.length(s) == 0 {
        return Err(ParseError.EmptyString)
    }
    // ... parsing logic with detailed errors
}
```

## Pattern 7: Higher-Order Function APIs

Accept functions for flexibility:

```kira
// Generic transformation
let transform_all[A, B]: fn(List[A], fn(A) -> B) -> List[B] = ...

// Generic filtering
let keep_where[A]: fn(List[A], fn(A) -> bool) -> List[A] = ...

// Generic reduction
let reduce[A, B]: fn(List[A], B, fn(B, A) -> B) -> B = ...

// Custom comparison
let sort_by[A]: fn(List[A], fn(A, A) -> Ordering) -> List[A] = ...

// Usage
let sorted_users: List[User] = sort_by[User](
    users,
    fn(a: User, b: User) -> Ordering {
        return compare_strings(a.name, b.name)
    }
)
```

## Pattern 8: Module Organization

Structure modules with clear public APIs:

```kira
module user

// Public types
pub type User = {
    id: UserId,
    name: string,
    email: Email
}

pub type UserError =
    | NotFound(UserId)
    | InvalidEmail(string)
    | DuplicateUser(string)

// Public API
pub let create: fn(string, string) -> Result[User, UserError] = ...
pub let find_by_id: fn(UserId) -> Option[User] = ...
pub let update_email: fn(User, string) -> Result[User, UserError] = ...

// Private helpers (not pub)
let validate_email: fn(string) -> Result[Email, UserError] = ...
let generate_id: fn() -> UserId = ...
```

## Pattern 9: Explicit Effect Boundaries

Mark IO boundaries clearly:

```kira
// Pure domain logic (no effect)
let validate_order: fn(Order) -> Result[ValidOrder, ValidationError] = ...
let calculate_total: fn(ValidOrder) -> Money = ...
let apply_tax: fn(Money, TaxRate) -> Money = ...

// Effectful IO operations
effect fn load_tax_rate(region: Region) -> IO[Result[TaxRate, Error]] = ...
effect fn save_order(order: Order) -> IO[Result[void, Error]] = ...
effect fn notify_customer(order: Order) -> IO[Result[void, Error]] = ...

// Composition at effect boundary
effect fn process_order(order: Order) -> IO[Result[ProcessedOrder, Error]] {
    let valid: ValidOrder = validate_order(order)?    // Pure
    let total: Money = calculate_total(valid)          // Pure
    let tax_rate: TaxRate = load_tax_rate(order.region)?  // Effect
    let final_total: Money = apply_tax(total, tax_rate)   // Pure
    save_order(order)?                                 // Effect
    notify_customer(order)?                            // Effect
    return Ok(ProcessedOrder { ... })
}
```

## Pattern 10: Documentation

Document all public items:

```kira
/// Creates a new user with the given name and email.
///
/// # Arguments
/// * `name` - The user's display name (1-100 characters)
/// * `email` - The user's email address (must be valid format)
///
/// # Returns
/// A new User on success, or UserError if validation fails.
///
/// # Errors
/// * `InvalidEmail` - The email format is invalid
/// * `DuplicateUser` - A user with this email already exists
///
/// # Example
/// ```
/// let user: Result[User, UserError] = create("Alice", "alice@example.com")
/// ```
pub let create: fn(string, string) -> Result[User, UserError] = ...
```

## API Design Checklist

- [ ] Pure and effectful code clearly separated
- [ ] Config records for 4+ parameters
- [ ] Smart constructors validate data
- [ ] Type-safe identifiers for different IDs
- [ ] Result returned for fallible operations
- [ ] Higher-order functions for flexibility
- [ ] Public API documented
- [ ] Private helpers not exposed
