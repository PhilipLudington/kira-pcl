---
globs: ["**/*.ki"]
---

# Naming Conventions

## Case Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Types | `PascalCase` | `Option`, `UserData`, `HttpClient` |
| Functions | `snake_case` | `create_user`, `parse_int` |
| Variables | `snake_case` | `user_count`, `is_valid` |
| Record fields | `snake_case` | `user_name`, `created_at` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_USERS`, `PI` |
| Sum type variants | `PascalCase` | `Some`, `None`, `NotFound` |
| Modules | `snake_case` | `std.list`, `std.option` |
| Type parameters | `T`, `PascalCase` | `T`, `Key`, `Value` |

## Function Naming Patterns

| Pattern | Meaning |
|---------|---------|
| `create` | Construct a new value |
| `from_*` / `to_*` | Type conversion |
| `parse_*` | Parse from string/data |
| `is_*` / `has_*` / `can_*` | Boolean query |
| `try_*` | May fail (returns Result) |
| `*_or` | Fallback variant |
| `map_*` | Transform contained value |
| `and_then` | Chain operations |

## Boolean Naming

Booleans MUST read as true/false statements:

```kira
// GOOD
let is_valid: bool = true
let has_items: bool = std.list.length(items) > 0
let can_proceed: fn(State) -> bool = fn(s: State) -> bool { ... }

// BAD
let valid: bool = true      // Ambiguous
let items: bool = true      // Noun
let proceed: fn() -> bool   // Verb, not query
```

Prefixes: `is_`, `has_`, `can_`, `should_`, `was_`, `will_`

## Acronyms

Treat acronyms as words:

```kira
// GOOD
type HttpClient = { ... }
let parse_json: fn(string) -> Result[Json, ParseError] = ...
let html_content: string = ...

// BAD
type HTTPClient = { ... }
let parseJSON: fn(...) = ...
```

## Abbreviations

Acceptable: `len`, `str`, `num`, `max`, `min`, `err`, `msg`, `ctx`

Avoid: cryptic abbreviations, single letters (except `i`, `j`, `n` for loops)
