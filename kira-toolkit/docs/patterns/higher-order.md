# Higher-Order Function Patterns

Patterns for effective use of higher-order functions in Kira.

## The Higher-Order Model

In Kira:
- Functions are first-class values
- Functions can be passed as arguments
- Functions can be returned from other functions
- Closures capture their environment immutably

## Pattern 1: Map - Transform Each Element

Use `map` to transform every element in a collection:

```kira
import std.list.{ map }

let numbers: List[i32] = Cons(1, Cons(2, Cons(3, Nil)))

// Double each number
let doubled: List[i32] = map[i32, i32](
    numbers,
    fn(x: i32) -> i32 { return x * 2 }
)
// Result: Cons(2, Cons(4, Cons(6, Nil)))

// Convert to strings
let strings: List[string] = map[i32, string](
    numbers,
    fn(x: i32) -> string { return x.to_string() }
)
// Result: Cons("1", Cons("2", Cons("3", Nil)))

// Extract field from records
let users: List[User] = ...
let names: List[string] = map[User, string](
    users,
    fn(u: User) -> string { return u.name }
)
```

## Pattern 2: Filter - Select Matching Elements

Use `filter` to keep only elements matching a predicate:

```kira
import std.list.{ filter }

let numbers: List[i32] = Cons(1, Cons(2, Cons(3, Cons(4, Cons(5, Nil)))))

// Keep even numbers
let evens: List[i32] = filter[i32](
    numbers,
    fn(x: i32) -> bool { return x % 2 == 0 }
)
// Result: Cons(2, Cons(4, Nil))

// Keep positive numbers
let positives: List[i32] = filter[i32](
    numbers,
    fn(x: i32) -> bool { return x > 0 }
)

// Filter by record field
let active_users: List[User] = filter[User](
    users,
    fn(u: User) -> bool { return u.is_active }
)
```

## Pattern 3: Fold - Reduce to Single Value

Use `fold` to combine all elements into one value:

```kira
import std.list.{ fold }

let numbers: List[i32] = Cons(1, Cons(2, Cons(3, Cons(4, Cons(5, Nil)))))

// Sum all numbers
let sum: i32 = fold[i32, i32](
    numbers,
    0,
    fn(acc: i32, x: i32) -> i32 { return acc + x }
)
// Result: 15

// Find maximum
let max: i32 = fold[i32, i32](
    numbers,
    i32.MIN,
    fn(acc: i32, x: i32) -> i32 {
        if x > acc { return x }
        return acc
    }
)
// Result: 5

// Build a string
let joined: string = fold[i32, string](
    numbers,
    "",
    fn(acc: string, x: i32) -> string {
        if std.string.length(acc) == 0 {
            return x.to_string()
        }
        return acc + ", " + x.to_string()
    }
)
// Result: "1, 2, 3, 4, 5"
```

## Pattern 4: Combine Operations

Chain map, filter, and fold for complex transformations:

```kira
// Get sum of squares of positive numbers
let sum_of_positive_squares: fn(List[i32]) -> i32 =
    fn(numbers: List[i32]) -> i32 {
        let positives: List[i32] = filter[i32](
            numbers,
            fn(x: i32) -> bool { return x > 0 }
        )
        let squares: List[i32] = map[i32, i32](
            positives,
            fn(x: i32) -> i32 { return x * x }
        )
        let sum: i32 = fold[i32, i32](
            squares,
            0,
            fn(acc: i32, x: i32) -> i32 { return acc + x }
        )
        return sum
    }
```

## Pattern 5: Function Composition

Build complex functions from simpler ones:

```kira
// Explicit composition function
let compose[A, B, C]: fn(fn(B) -> C, fn(A) -> B) -> fn(A) -> C =
    fn(f: fn(B) -> C, g: fn(A) -> B) -> fn(A) -> C {
        return fn(x: A) -> C {
            return f(g(x))
        }
    }

// Usage
let add_one: fn(i32) -> i32 = fn(x: i32) -> i32 { return x + 1 }
let double: fn(i32) -> i32 = fn(x: i32) -> i32 { return x * 2 }
let square: fn(i32) -> i32 = fn(x: i32) -> i32 { return x * x }

// Compose: square(double(add_one(x)))
let transform: fn(i32) -> i32 = compose[i32, i32, i32](
    square,
    compose[i32, i32, i32](double, add_one)
)

let result: i32 = transform(5)  // square(double(6)) = square(12) = 144
```

## Pattern 6: Partial Application

Create specialized functions from general ones:

```kira
// General function
let add: fn(i32, i32) -> i32 = fn(a: i32, b: i32) -> i32 {
    return a + b
}

// Partial application factory
let partial_add: fn(i32) -> fn(i32) -> i32 = fn(a: i32) -> fn(i32) -> i32 {
    return fn(b: i32) -> i32 { return a + b }
}

// Create specialized functions
let add_ten: fn(i32) -> i32 = partial_add(10)
let add_five: fn(i32) -> i32 = partial_add(5)

let result1: i32 = add_ten(3)   // 13
let result2: i32 = add_five(3)  // 8
```

## Pattern 7: Closures for State

Capture state in closures:

```kira
// Counter factory
let make_counter: fn(i32) -> fn() -> i32 = fn(start: i32) -> fn() -> i32 {
    // Note: In pure Kira, this would need a different approach
    // since we can't mutate captured variables
    // This is conceptual
    return fn() -> i32 { return start }
}

// Accumulator pattern (pure version)
let make_accumulator: fn(i32) -> fn(i32) -> i32 = fn(initial: i32) -> fn(i32) -> i32 {
    return fn(value: i32) -> i32 { return initial + value }
}

let add_from_100: fn(i32) -> i32 = make_accumulator(100)
let result: i32 = add_from_100(5)  // 105
```

## Pattern 8: Higher-Order Predicates

Build complex predicates from simple ones:

```kira
// Predicate combinators
let and_pred[T]: fn(fn(T) -> bool, fn(T) -> bool) -> fn(T) -> bool =
    fn(p1: fn(T) -> bool, p2: fn(T) -> bool) -> fn(T) -> bool {
        return fn(x: T) -> bool { return p1(x) and p2(x) }
    }

let or_pred[T]: fn(fn(T) -> bool, fn(T) -> bool) -> fn(T) -> bool =
    fn(p1: fn(T) -> bool, p2: fn(T) -> bool) -> fn(T) -> bool {
        return fn(x: T) -> bool { return p1(x) or p2(x) }
    }

let not_pred[T]: fn(fn(T) -> bool) -> fn(T) -> bool =
    fn(p: fn(T) -> bool) -> fn(T) -> bool {
        return fn(x: T) -> bool { return not p(x) }
    }

// Usage
let is_positive: fn(i32) -> bool = fn(x: i32) -> bool { return x > 0 }
let is_even: fn(i32) -> bool = fn(x: i32) -> bool { return x % 2 == 0 }

let is_positive_even: fn(i32) -> bool = and_pred[i32](is_positive, is_even)
let is_odd: fn(i32) -> bool = not_pred[i32](is_even)
```

## Pattern 9: Find and Any/All

Search operations using predicates:

```kira
import std.list.{ find, any, all }

let numbers: List[i32] = Cons(1, Cons(2, Cons(3, Cons(4, Cons(5, Nil)))))

// Find first even number
let first_even: Option[i32] = find[i32](
    numbers,
    fn(x: i32) -> bool { return x % 2 == 0 }
)
// Result: Some(2)

// Check if any number is greater than 4
let has_large: bool = any[i32](
    numbers,
    fn(x: i32) -> bool { return x > 4 }
)
// Result: true

// Check if all numbers are positive
let all_positive: bool = all[i32](
    numbers,
    fn(x: i32) -> bool { return x > 0 }
)
// Result: true
```

## Pattern 10: Result/Option Chaining

Use higher-order functions on Result and Option:

```kira
import std.result.{ map as map_result, and_then }
import std.option.{ map as map_option }

// Map over Result
let parse_and_double: fn(string) -> Result[i32, ParseError] =
    fn(s: string) -> Result[i32, ParseError] {
        let parsed: Result[i32, ParseError] = parse_int(s)
        return map_result[i32, ParseError, i32](
            parsed,
            fn(n: i32) -> i32 { return n * 2 }
        )
    }

// Chain Result operations
let parse_and_validate: fn(string) -> Result[i32, Error] =
    fn(s: string) -> Result[i32, Error] {
        let parsed: Result[i32, Error] = parse_int(s)
        return and_then[i32, Error, i32](
            parsed,
            fn(n: i32) -> Result[i32, Error] {
                if n > 0 {
                    return Ok(n)
                }
                return Err(Error.NegativeNumber)
            }
        )
    }
```

## Higher-Order Function Checklist

- [ ] Use map for transforming collections
- [ ] Use filter for selecting elements
- [ ] Use fold for reducing to single value
- [ ] Chain operations for complex transformations
- [ ] Use partial application for specialization
- [ ] Build complex predicates from simple ones
- [ ] Use find/any/all for search operations
