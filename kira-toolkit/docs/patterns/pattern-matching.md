# Pattern Matching Patterns

Comprehensive patterns for safe and expressive pattern matching in Kira.

## The Pattern Matching Model

Kira's pattern matching:
- Is statement-based (assigns to variables)
- Must be exhaustive (all cases covered)
- Supports nested patterns
- Allows guards for conditional matching

## Pattern 1: Exhaustive Matching on Sum Types

Always handle all variants of a sum type:

```kira
type Result[T, E] =
    | Ok(T)
    | Err(E)

let handle_result: fn(Result[i32, string]) -> i32 =
    fn(r: Result[i32, string]) -> i32 {
        var output: i32
        match r {
            Ok(value) => { output = value }
            Err(msg) => { output = 0 }  // Handle error case
        }
        return output
    }
```

## Pattern 2: Nested Pattern Matching

Match deeply nested structures in one pattern:

```kira
type Response =
    | Success { data: Option[User] }
    | Error { code: i32 }

let extract_username: fn(Response) -> Option[string] =
    fn(resp: Response) -> Option[string] {
        var result: Option[string]
        match resp {
            Success { data: Some(user) } => { result = Some(user.name) }
            Success { data: None } => { result = None }
            Error { code: _ } => { result = None }
        }
        return result
    }
```

## Pattern 3: Guard Patterns

Use guards for conditional matching:

```kira
let categorize: fn(i32) -> string = fn(n: i32) -> string {
    var result: string
    match n {
        x if x < 0 => { result = "negative" }
        0 => { result = "zero" }
        x if x < 10 => { result = "small positive" }
        x if x < 100 => { result = "medium" }
        _ => { result = "large" }
    }
    return result
}
```

## Pattern 4: Or Patterns

Match multiple values with the same handler:

```kira
let is_vowel: fn(char) -> bool = fn(c: char) -> bool {
    var result: bool
    match c {
        'a' | 'e' | 'i' | 'o' | 'u' => { result = true }
        'A' | 'E' | 'I' | 'O' | 'U' => { result = true }
        _ => { result = false }
    }
    return result
}

let day_type: fn(string) -> string = fn(day: string) -> string {
    var result: string
    match day {
        "Saturday" | "Sunday" => { result = "weekend" }
        "Monday" | "Tuesday" | "Wednesday" | "Thursday" | "Friday" => { result = "weekday" }
        _ => { result = "invalid" }
    }
    return result
}
```

## Pattern 5: Record Pattern Matching

Destructure record fields in patterns:

```kira
type Point = { x: f64, y: f64 }

let describe_point: fn(Point) -> string = fn(p: Point) -> string {
    var result: string
    match p {
        Point { x: 0.0, y: 0.0 } => { result = "origin" }
        Point { x: 0.0, y: y } => { result = "on y-axis at {y}" }
        Point { x: x, y: 0.0 } => { result = "on x-axis at {x}" }
        Point { x: x, y: y } if x == y => { result = "on diagonal at {x}" }
        Point { x: x, y: y } => { result = "at ({x}, {y})" }
    }
    return result
}
```

## Pattern 6: Tuple Pattern Matching

Destructure tuples in patterns:

```kira
let compare: fn((i32, i32)) -> string = fn(pair: (i32, i32)) -> string {
    var result: string
    match pair {
        (a, b) if a == b => { result = "equal" }
        (a, b) if a < b => { result = "first smaller" }
        (a, b) => { result = "first larger" }
    }
    return result
}
```

## Pattern 7: Recursive Data Structure Matching

Match on recursive types like List:

```kira
type List[T] =
    | Cons(T, List[T])
    | Nil

let sum: fn(List[i32]) -> i32 = fn(list: List[i32]) -> i32 {
    var result: i32
    match list {
        Nil => { result = 0 }
        Cons(head, tail) => { result = head + sum(tail) }
    }
    return result
}

let take: fn(List[i32], i32) -> List[i32] = fn(list: List[i32], n: i32) -> List[i32] {
    var result: List[i32]
    match (list, n) {
        (_, 0) => { result = Nil }
        (Nil, _) => { result = Nil }
        (Cons(head, tail), n) => { result = Cons(head, take(tail, n - 1)) }
    }
    return result
}
```

## Pattern 8: Let Destructuring

Use patterns in let bindings:

```kira
// Destructure tuple
let pair: (i32, string) = (42, "hello")
let (num, text): (i32, string) = pair

// Destructure record
let point: Point = Point { x: 3.0, y: 4.0 }
let Point { x: px, y: py }: Point = point

// Destructure nested
let response: (bool, Option[i32]) = (true, Some(42))
let (success, value): (bool, Option[i32]) = response
```

## Pattern 9: Wildcard Patterns

Use `_` for values you don't need:

```kira
let is_ok: fn(Result[i32, string]) -> bool = fn(r: Result[i32, string]) -> bool {
    var result: bool
    match r {
        Ok(_) => { result = true }   // Don't need the value
        Err(_) => { result = false } // Don't need the error
    }
    return result
}

let first_element: fn(List[i32]) -> Option[i32] = fn(list: List[i32]) -> Option[i32] {
    var result: Option[i32]
    match list {
        Cons(head, _) => { result = Some(head) }  // Don't need tail
        Nil => { result = None }
    }
    return result
}
```

## Pattern 10: Combining Patterns

Build complex matches from simpler patterns:

```kira
type Command =
    | Move { x: i32, y: i32 }
    | Rotate { angle: f64 }
    | Scale { factor: f64 }
    | Sequence(List[Command])

let describe_command: fn(Command) -> string = fn(cmd: Command) -> string {
    var result: string
    match cmd {
        Move { x: 0, y: 0 } => { result = "no movement" }
        Move { x: x, y: 0 } => { result = "horizontal move by {x}" }
        Move { x: 0, y: y } => { result = "vertical move by {y}" }
        Move { x: x, y: y } => { result = "diagonal move to ({x}, {y})" }
        Rotate { angle: a } if a == 0.0 => { result = "no rotation" }
        Rotate { angle: a } => { result = "rotate by {a} degrees" }
        Scale { factor: 1.0 } => { result = "no scaling" }
        Scale { factor: f } => { result = "scale by {f}x" }
        Sequence(Nil) => { result = "empty sequence" }
        Sequence(Cons(_, _)) => { result = "command sequence" }
    }
    return result
}
```

## Anti-Patterns to Avoid

### 1. Non-Exhaustive Matches

```kira
// BAD: Missing case (compile error in Kira)
// match option {
//     Some(x) => { ... }
//     // Missing None case!
// }
```

### 2. Catch-All Before Specific Patterns

```kira
// BAD: Specific patterns unreachable
// match n {
//     _ => { result = "other" }  // Catches everything!
//     0 => { result = "zero" }   // Never reached
// }
```

### 3. Overly Nested Matches

```kira
// BAD: Deeply nested matches
// match a {
//     Some(x) => {
//         match x {
//             Ok(y) => {
//                 match y { ... }
//             }
//         }
//     }
// }

// GOOD: Use nested patterns
match a {
    Some(Ok(y)) => { ... }
    Some(Err(e)) => { ... }
    None => { ... }
}
```

## Pattern Matching Checklist

- [ ] All match statements are exhaustive
- [ ] Specific patterns come before general ones
- [ ] Nested patterns used instead of nested matches
- [ ] Wildcards used for unused bindings
- [ ] Guards used for conditional matching
