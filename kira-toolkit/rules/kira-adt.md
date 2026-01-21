---
globs: ["**/*.ki"]
---

# Algebraic Data Type Rules

## Sum Types (Tagged Unions)

Use sum types for values that can be one of several variants:

```kira
// Basic sum type
type Option[T] =
    | Some(T)
    | None

// Sum type with multiple data
type Result[T, E] =
    | Ok(T)
    | Err(E)

// Sum type with named fields
type HttpError =
    | NotFound(string)
    | Unauthorized { reason: string }
    | ServerError { code: i32, message: string }

// Recursive sum type
type List[T] =
    | Cons(T, List[T])
    | Nil
```

**ADT1**: Use sum types when a value can be one of several distinct variants.

## Product Types (Records)

Use product types for values that have multiple named fields:

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

**ADT2**: Use product types when a value needs multiple named fields.

## Generic Types

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
```

## Make Illegal States Unrepresentable

**ADT3**: Design types so invalid states cannot be constructed:

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

## Prefer Specific Types

**ADT4**: Prefer many specific types over few general types:

```kira
// BAD: Overly general
type Event = {
    event_type: string,
    data: string
}

// GOOD: Specific variants
type Event =
    | UserCreated { user_id: i64, name: string }
    | UserDeleted { user_id: i64 }
    | LoginAttempt { user_id: i64, success: bool }
```

## Type Aliases

```kira
// Simple alias for clarity
type UserId = i64
type Email = string

// Generic alias
type StringResult[T] = Result[T, string]
type IntList = List[i32]
```

## Record Updates

Create new records with modified fields:

```kira
let with_name: fn(User, string) -> User = fn(user: User, name: string) -> User {
    return User {
        id: user.id,
        name: name,
        email: user.email,
        created_at: user.created_at
    }
}
```

## ADT Design Checklist

- [ ] Sum types used for "one of" relationships
- [ ] Product types used for "all of" relationships
- [ ] Invalid states are unrepresentable
- [ ] Types are specific, not overly general
- [ ] Generic types have meaningful type parameters
