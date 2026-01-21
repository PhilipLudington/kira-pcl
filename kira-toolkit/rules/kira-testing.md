---
globs: ["**/*.ki", "**/tests/**"]
---

# Testing Rules

## Test Organization

**T1**: One test file per module: `tests/test_<module>.ki`

**T2**: Test naming: `test_<function>_<scenario>`

```kira
// tests/test_user.ki

let test_create_user_success: fn() -> void = fn() -> void {
    let user: Result[User, UserError] = create_user("Alice")
    assert(is_ok(user))
    return
}

let test_create_user_empty_name: fn() -> void = fn() -> void {
    let result: Result[User, UserError] = create_user("")
    assert(is_err(result))
    return
}

let test_create_user_name_too_long: fn() -> void = fn() -> void {
    let long_name: string = "a".repeat(100)
    let result: Result[User, UserError] = create_user(long_name)
    assert(is_err(result))
    return
}
```

## Required Test Categories

Every module needs tests for:

1. **Happy path** - Normal successful operation
2. **Edge cases** - Empty input, zero values, boundaries
3. **Error conditions** - Invalid input, missing resources
4. **Property tests** - Invariants that should always hold

## Test Structure

Use Arrange-Act-Assert:

```kira
let test_map_doubles_values: fn() -> void = fn() -> void {
    // Arrange
    let input: List[i32] = Cons(1, Cons(2, Cons(3, Nil)))

    // Act
    let result: List[i32] = map[i32, i32](input, fn(x: i32) -> i32 { return x * 2 })

    // Assert
    assert(list_equals(result, Cons(2, Cons(4, Cons(6, Nil)))))
    return
}
```

## Testing Pure Functions

**T3**: Pure functions are easy to test - no mocks needed:

```kira
let test_factorial: fn() -> void = fn() -> void {
    assert(factorial(0) == 1)
    assert(factorial(1) == 1)
    assert(factorial(5) == 120)
    assert(factorial(10) == 3628800)
    return
}

let test_add_is_commutative: fn() -> void = fn() -> void {
    assert(add(3, 5) == add(5, 3))
    assert(add(0, 10) == add(10, 0))
    return
}
```

## Property-Based Testing

**T4**: Test properties that should always hold:

```kira
// Property: map preserves length
let test_map_preserves_length: fn() -> void = fn() -> void {
    for _ in range(0, 100) {
        let input: List[i32] = random_list()
        let output: List[i32] = map[i32, i32](input, fn(x: i32) -> i32 { return x * 2 })
        assert(length(input) == length(output))
    }
    return
}

// Property: filter result is subset
let test_filter_subset: fn() -> void = fn() -> void {
    for _ in range(0, 100) {
        let input: List[i32] = random_list()
        let output: List[i32] = filter[i32](input, fn(x: i32) -> bool { return x > 0 })
        assert(length(output) <= length(input))
    }
    return
}

// Property: fold with (+) equals sum
let test_fold_sum: fn() -> void = fn() -> void {
    for _ in range(0, 100) {
        let input: List[i32] = random_list()
        let folded: i32 = fold[i32, i32](input, 0, fn(a: i32, b: i32) -> i32 { return a + b })
        assert(folded == sum(input))
    }
    return
}
```

## Testing Effect Functions

**T5**: Effect functions need IO testing infrastructure:

```kira
effect fn test_read_file_not_found() -> IO[void] {
    let result: Result[string, IoError] = std.fs.read_file("/nonexistent")
    assert(is_err(result))
    return
}

effect fn test_write_then_read() -> IO[void] {
    let path: string = "/tmp/test_file.txt"
    let content: string = "test content"

    std.fs.write_file(path, content)?
    let read_content: string = std.fs.read_file(path)?

    assert(content == read_content)
    std.fs.remove(path)?
    return
}
```

## Assertions

**T6**: Use descriptive assertion messages:

```kira
assert(result.is_ok(), "Expected successful parse")
assert(value == expected, "Health should be {expected}, got {value}")
```

## Coverage Requirements

**T7**: All public API functions MUST have tests

**T8**: Error paths MUST be tested, not just happy paths

## Testing Checklist

- [ ] One test file per module
- [ ] Happy path tests exist
- [ ] Edge case tests exist
- [ ] Error condition tests exist
- [ ] Property-based tests for pure functions
- [ ] All public functions have tests
