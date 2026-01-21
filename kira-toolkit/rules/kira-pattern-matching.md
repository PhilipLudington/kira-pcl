---
globs: ["**/*.ki"]
---

# Pattern Matching Rules

## Exhaustiveness

**PM1**: Pattern matches MUST be exhaustive - all possible values must be covered.

```kira
// GOOD: All variants covered
let describe: fn(Option[i32]) -> string = fn(opt: Option[i32]) -> string {
    var result: string
    match opt {
        Some(n) => { result = "has value: {n}" }
        None => { result = "empty" }
    }
    return result
}

// BAD: Non-exhaustive (compile error)
// match opt {
//     Some(n) => { result = "value" }
//     // Missing None case!
// }
```

## Pattern Types

### Literal Patterns

```kira
match value {
    0 => { result = "zero" }
    1 => { result = "one" }
    _ => { result = "other" }
}
```

### Constructor Patterns (Sum Types)

```kira
match option {
    Some(x) => { result = x }
    None => { result = 0 }
}

match result_value {
    Ok(value) => { output = value }
    Err(e) => { output = handle_error(e) }
}
```

### Record Patterns (Product Types)

```kira
match point {
    Point { x: 0, y: 0 } => { result = "origin" }
    Point { x: 0, y: y } => { result = "on y-axis at {y}" }
    Point { x: x, y: 0 } => { result = "on x-axis at {x}" }
    Point { x: x, y: y } => { result = "at ({x}, {y})" }
}
```

### Tuple Patterns

```kira
match pair {
    (0, 0) => { result = "origin" }
    (x, 0) => { result = "x = {x}" }
    (0, y) => { result = "y = {y}" }
    (x, y) => { result = "({x}, {y})" }
}
```

### Or Patterns

```kira
match value {
    1 | 2 | 3 => { result = "small" }
    4 | 5 | 6 => { result = "medium" }
    _ => { result = "other" }
}
```

## Guard Patterns

```kira
match value {
    n if n < 0 => { result = "negative" }
    n if n > 100 => { result = "large" }
    n => { result = "normal: {n}" }
}
```

**PM2**: Place more specific patterns before general ones.

## Nested Patterns

```kira
match nested {
    Some(Ok(value)) => { result = value }
    Some(Err(e)) => { result = handle_error(e) }
    None => { result = default_value }
}
```

**PM3**: Prefer nested patterns over nested match statements.

## Destructuring in Let

```kira
// Destructure records
let p: Point = Point { x: 3.0, y: 4.0 }
let Point { x: px, y: py }: Point = p

// Destructure tuples
let pair: (i32, string) = (42, "hello")
let (n, s): (i32, string) = pair
```

## Wildcard Pattern

Use `_` for values you don't need:

```kira
match result {
    Ok(value) => { output = value }
    Err(_) => { output = default }  // Don't need error details
}
```

## Pattern Matching Checklist

- [ ] All match statements are exhaustive
- [ ] More specific patterns come before general ones
- [ ] Use `_` for unused bindings
- [ ] Prefer nested patterns over nested matches
- [ ] Guards are used for conditional matching
