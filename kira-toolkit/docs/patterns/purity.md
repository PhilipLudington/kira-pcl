# Purity and Effects Patterns

Patterns for working with pure and effectful code in Kira.

## The Purity Model

Kira distinguishes between:
- **Pure functions**: No side effects, deterministic, referentially transparent
- **Effect functions**: Can perform IO, mutation, or other side effects

## Pattern 1: Pure Core, Effectful Shell

Structure your application with pure business logic and effectful IO at the boundaries:

```kira
// Pure core - business logic
let calculate_total: fn(List[Item]) -> f64 = fn(items: List[Item]) -> f64 {
    return fold[Item, f64](
        items,
        0.0,
        fn(acc: f64, item: Item) -> f64 { return acc + item.price }
    )
}

let apply_discount: fn(f64, f64) -> f64 = fn(total: f64, discount: f64) -> f64 {
    return total * (1.0 - discount)
}

// Effectful shell - IO operations
effect fn process_order(order_id: i64) -> IO[Result[Receipt, Error]] {
    // IO: Fetch from database
    let items: List[Item] = fetch_items(order_id)?

    // Pure: Calculate total
    let total: f64 = calculate_total(items)
    let final_total: f64 = apply_discount(total, 0.1)

    // IO: Save receipt
    let receipt: Receipt = save_receipt(order_id, final_total)?

    // IO: Send notification
    send_notification(receipt)?

    return Ok(receipt)
}
```

Benefits:
- Pure functions are easy to test
- Business logic is isolated from IO concerns
- Pure code can be safely refactored

## Pattern 2: Effect Function Composition

Chain effectful operations using Result:

```kira
effect fn full_pipeline() -> IO[Result[Output, Error]] {
    let step1: Data1 = fetch_data()?
    let step2: Data2 = process(step1)?   // Pure transformation
    let step3: Data3 = validate(step2)?  // Pure validation
    let result: Output = save_result(step3)?
    return Ok(result)
}
```

## Pattern 3: Pure Data Transformation Pipelines

Build complex transformations from simple pure functions:

```kira
// Each step is pure
let normalize: fn(string) -> string = fn(s: string) -> string {
    return std.string.trim(std.string.to_lower(s))
}

let validate_length: fn(string) -> Result[string, Error] = fn(s: string) -> Result[string, Error] {
    if std.string.length(s) > 100 {
        return Err(Error.TooLong)
    }
    return Ok(s)
}

let sanitize: fn(string) -> string = fn(s: string) -> string {
    return remove_special_chars(s)
}

// Compose into pipeline
let process_input: fn(string) -> Result[string, Error] = fn(input: string) -> Result[string, Error] {
    let normalized: string = normalize(input)
    let validated: string = validate_length(normalized)?
    let sanitized: string = sanitize(validated)
    return Ok(sanitized)
}
```

## Pattern 4: Testing Pure Functions

Pure functions are trivial to test - no mocks needed:

```kira
// Function to test
let add: fn(i32, i32) -> i32 = fn(a: i32, b: i32) -> i32 {
    return a + b
}

// Tests are simple assertions
let test_add_positive: fn() -> void = fn() -> void {
    assert(add(2, 3) == 5)
    return
}

let test_add_negative: fn() -> void = fn() -> void {
    assert(add(-2, 3) == 1)
    return
}

let test_add_commutative: fn() -> void = fn() -> void {
    assert(add(2, 3) == add(3, 2))
    return
}
```

## Pattern 5: Memoization of Pure Functions

Pure functions can be safely memoized:

```kira
// Expensive pure computation
let fibonacci: fn(i32) -> i32 = fn(n: i32) -> i32 {
    if n <= 1 {
        return n
    }
    return fibonacci(n - 1) + fibonacci(n - 2)
}

// Memoized version (conceptual - would need runtime support)
let memoized_fib: fn(i32) -> i32 = memoize(fibonacci)
```

## Pattern 6: Effect Isolation

Keep effect boundaries explicit and minimal:

```kira
// BAD: Effects scattered throughout
effect fn bad_process() -> IO[void] {
    std.log.info("Starting")           // Effect
    let a: i32 = compute_a()           // Pure
    std.log.info("Computed a")         // Effect
    let b: i32 = compute_b(a)          // Pure
    std.log.info("Computed b")         // Effect
    return
}

// GOOD: Effects at boundaries only
let compute_all: fn() -> (i32, i32) = fn() -> (i32, i32) {
    let a: i32 = compute_a()
    let b: i32 = compute_b(a)
    return (a, b)
}

effect fn good_process() -> IO[void] {
    std.log.info("Starting computation")
    let (a, b): (i32, i32) = compute_all()  // All pure work
    std.log.info("Computation complete: a={a}, b={b}")
    return
}
```

## Anti-Patterns to Avoid

### 1. Hidden Effects in "Pure" Functions

```kira
// BAD: Looks pure but does IO (would be compile error)
// let bad_fn: fn(i32) -> i32 = fn(x: i32) -> i32 {
//     std.log.debug("called with {x}")  // ERROR: Cannot do IO here
//     return x * 2
// }
```

### 2. Global Mutable State

```kira
// BAD: Mutable global state breaks purity guarantees
// In Kira, this pattern is not available in pure code
```

### 3. Mixing Pure and Effect Logic

```kira
// BAD: Business logic mixed with IO
effect fn process_items(items: List[Item]) -> IO[List[Result]] {
    var results: List[Result] = Nil
    for item in items {
        std.log.debug("Processing {item}")  // Logging mixed with logic
        let processed: Result = transform(item)
        results = Cons(processed, results)
    }
    return results
}

// GOOD: Separate concerns
let process_all: fn(List[Item]) -> List[Result] = fn(items: List[Item]) -> List[Result] {
    return map[Item, Result](items, transform)
}

effect fn process_items_logged(items: List[Item]) -> IO[List[Result]] {
    std.log.debug("Processing {length(items)} items")
    let results: List[Result] = process_all(items)
    std.log.debug("Processing complete")
    return results
}
```

## Purity Checklist

- [ ] Business logic is in pure functions
- [ ] IO operations are in effect functions
- [ ] Effect boundaries are explicit
- [ ] Pure functions don't access global state
- [ ] Tests use pure functions directly (no mocks)
