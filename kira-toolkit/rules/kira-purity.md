---
globs: ["**/*.ki"]
---

# Purity and Effects Rules

## Pure Functions (Default)

All functions without `effect` keyword are pure. Pure functions cannot:
- Perform IO (print, read files, network)
- Mutate global state
- Generate random numbers
- Get current time
- Call effectful functions

```kira
// Pure function - no side effects allowed
let add: fn(i32, i32) -> i32 = fn(a: i32, b: i32) -> i32 {
    return a + b
}

let factorial: fn(i32) -> i32 = fn(n: i32) -> i32 {
    if n <= 1 {
        return 1
    }
    return n * factorial(n - 1)
}
```

## Effect Functions

Use `effect fn` for functions that perform side effects:

```kira
// Effect function - can perform IO
effect fn print_greeting(name: string) -> IO[void] {
    std.io.println("Hello, {name}!")
    return
}

// Effect function with error handling
effect fn read_config(path: string) -> IO[Result[Config, Error]] {
    let content: Result[string, IoError] = std.fs.read_file(path)
    match content {
        Ok(s) => { return Ok(parse_config(s)) }
        Err(e) => { return Err(Error.Io(e)) }
    }
}
```

## Effect Propagation Rules

**P1**: Effect functions can call both pure and effectful code:

```kira
effect fn process(x: i32) -> IO[i32] {
    let doubled: i32 = x * 2         // OK: pure computation
    std.io.println("processing")      // OK: IO in IO context
    return doubled
}
```

**P2**: Pure functions CANNOT call effectful code (compile error):

```kira
// This would be a compile error:
// let bad: fn(i32) -> i32 = fn(x: i32) -> i32 {
//     std.io.println("hello")  // ERROR: pure function cannot do IO
//     return x
// }
```

## Effect Types

| Effect Type | Meaning |
|-------------|---------|
| `IO[T]` | Function performs IO, returns T |
| `IO[Result[T, E]]` | IO that may fail with error E |
| `IO[Option[T]]` | IO that may return nothing |

## Main Function

Main is always an effect function:

```kira
effect fn main() -> IO[void] {
    let result: i32 = factorial(5)   // OK: calling pure from effect
    std.io.println("Result: {result}")
    return
}
```

## Purity Benefits

Pure functions enable:
- Safe parallelization
- Memoization
- Equational reasoning
- Easy testing (no mocks needed)
- Refactoring confidence

## Purity Checklist

- [ ] Pure functions don't call effectful functions
- [ ] Effect functions are marked with `effect fn`
- [ ] Effect return types reflect what can happen (`IO[T]`)
- [ ] Main function is properly marked as effect
