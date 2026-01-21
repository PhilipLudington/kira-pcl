# Kira-Toolkit Quick Reference

> For Claude Code and AI-assisted Kira development
>
> **Version 0.1.0** - Initial Release

## Standards Location

Full standards: `STANDARDS.md`

## Key Rules

### Naming
- Types: `PascalCase` (Option, Result, UserData)
- Functions: `snake_case` (create_user, parse_int)
- Variables: `snake_case` (user_count, is_valid)
- Constants: `UPPER_SNAKE_CASE` (MAX_SIZE, DEFAULT_PORT)
- Modules: `snake_case` (std.list, std.option)

### Purity and Effects
- All functions are pure by default (no IO, no mutation)
- Use `effect fn` for functions that perform IO
- Effects are visible in return types: `IO[T]`, `IO[Result[T, E]]`
- Pure code cannot call effectful code

### Syntax Requirements
- **Explicit types** on all bindings: `let x: i32 = 42`
- **Explicit return** in all functions: `return value`
- **Statement-based control flow**: assign inside blocks
- **Closures with full types**: `fn(x: i32) -> i32 { return x * 2 }`

### Algebraic Data Types
- **Sum types**: `type Option[T] = | Some(T) | None`
- **Product types**: `type Point = { x: f64, y: f64 }`
- Use sum types for variants, product types for records

### Pattern Matching
- All patterns must be exhaustive
- Use wildcard `_` for catch-all
- Use guards with `if`: `n if n > 0 => { ... }`

### Error Handling
- Return `Result[T, E]` for fallible operations
- Use `?` to propagate errors
- Use `Option[T]` for values that may be absent
- Use `??` for default values: `value ?? default`

### Higher-Order Functions
- Functions are first-class values
- Use `map`, `filter`, `fold` from std.list
- Closures capture environment immutably

### Common Patterns

```kira
// Pure function
let add: fn(i32, i32) -> i32 = fn(a: i32, b: i32) -> i32 {
    return a + b
}

// Effect function
effect fn main() -> IO[void] {
    let result: i32 = add(2, 3)
    std.io.println("Result: {result}")
    return
}

// Sum type and pattern matching
type Shape =
    | Circle(f64)
    | Rectangle(f64, f64)

let area: fn(Shape) -> f64 = fn(shape: Shape) -> f64 {
    var result: f64
    match shape {
        Circle(r) => { result = 3.14159 * r * r }
        Rectangle(w, h) => { result = w * h }
    }
    return result
}

// Error handling with Result
effect fn read_config(path: string) -> IO[Result[Config, Error]] {
    let content: Result[string, IoError] = std.fs.read_file(path)
    match content {
        Ok(s) => { return Ok(parse_config(s)) }
        Err(e) => { return Err(Error.Io(e)) }
    }
}

// Higher-order function usage
let numbers: List[i32] = Cons(1, Cons(2, Cons(3, Nil)))
let doubled: List[i32] = std.list.map[i32, i32](
    numbers,
    fn(x: i32) -> i32 { return x * 2 }
)

// Function composition
let process: fn(i32) -> i32 = fn(x: i32) -> i32 {
    let step1: i32 = add_one(x)
    let step2: i32 = double(step1)
    return step2
}
```

## Checklist

Before committing:
- [ ] Types are `PascalCase`, functions are `snake_case`
- [ ] All bindings have explicit type annotations
- [ ] All functions use explicit `return` statements
- [ ] Pure functions don't call effectful functions
- [ ] Pattern matches are exhaustive
- [ ] Fallible operations return `Result[T, E]`
- [ ] External input is validated
- [ ] Public items are documented
- [ ] Tests exist for public functions

## Commands

| Command | Purpose |
|---------|---------|
| `/kira-init` | Create new Kira-Toolkit project |
| `/kira-install` | Add Kira-Toolkit to existing project |
| `/kira-review` | Review code against standards |
| `/kira-safety` | Security-focused review |
| `/kira-check` | Run build, tests, and checks |
| `/kira-update` | Update Kira-Toolkit |

## Rule Categories

Rules are auto-loaded from `.claude/rules/`:
- `kira-naming.md` - Naming conventions
- `kira-purity.md` - Pure vs effect functions
- `kira-pattern-matching.md` - Exhaustive matching
- `kira-adt.md` - Algebraic data type design
- `kira-errors.md` - Result/Option patterns
- `kira-api-design.md` - API design patterns
- `kira-testing.md` - Testing standards
- `kira-logging.md` - Logging standards
- `kira-security.md` - Security practices
