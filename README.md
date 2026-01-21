# kira-pcl

A Parser Combinator Library for [Kira](https://github.com/kira-lang/kira) that demonstrates functional programming patterns including algebraic data types, generics, higher-order functions, and effect-based IO.

## Features

- **Type-Safe Parsing**: Generic `Parser[T]` type with `ParseResult[T]` sum type
- **Rich Combinators**: `and_then`, `or_else`, `many`, `sep_by`, `chainl1`, and more
- **Lexer Helpers**: Whitespace handling, identifiers, numbers, string literals
- **Expression Parsing**: Operator precedence with left/right associativity
- **Error Reporting**: Position tracking with line/column and context stack
- **Pure Core**: Parsing logic is pure; IO isolated to boundaries

## Module Organization

```
src/
  types.ki       # Core types: Position, Input, ParseResult, ParseError
  primitives.ki  # Basic parsers: char_p, string_p, digit, letter
  combinators.ki # Combinators: and_then, or_else, many, map
  lexer.ki       # Token parsing: identifier, integer, string_literal
  expr.ki        # Expression parsing: chainl1, build_expr_parser
  io.ki          # File parsing and error formatting
  pcl.ki         # Main module (re-exports all public API)
```

## Quick Start

```kira
import src.pcl

// Parse an identifier
let id_parser: Parser[string] = identifier()
let result: Result[string, ParseError] = parse[string](id_parser, "foo123")
// => Ok("foo123")

// Parse comma-separated integers
let nums: Parser[List[i64]] = comma_sep[i64](integer())
let result: Result[List[i64], ParseError] = parse[List[i64]](nums, "1, 2, 3")
// => Ok(Cons(1, Cons(2, Cons(3, Nil))))

// Combine parsers
let pair_parser: Parser[(string, i64)] = map2[string, i64, (string, i64)](
    identifier(),
    seq_right[string, i64](symbol("="), integer()),
    fn(name: string, value: i64) -> (string, i64) { return (name, value) }
)
let result: Result[(string, i64), ParseError] = parse[(string, i64)](pair_parser, "x = 42")
// => Ok(("x", 42))
```

## Core Types

```kira
/// Parse result - success with value and remaining input, or failure with error
type ParseResult[T] =
    | Success { value: T, remaining: Input }
    | Failure { error: ParseError }

/// Parser is a function from input to parse result
type Parser[T] = fn(Input) -> ParseResult[T]

/// Position in source for error reporting
type Position = {
    offset: i64,
    line: i32,
    column: i32
}
```

## Combinator Reference

### Sequencing

| Combinator | Signature | Description |
|------------|-----------|-------------|
| `and_then` | `fn(Parser[A], fn(A) -> Parser[B]) -> Parser[B]` | Monadic bind |
| `seq_left` | `fn(Parser[A], Parser[B]) -> Parser[A]` | Parse both, keep left |
| `seq_right` | `fn(Parser[A], Parser[B]) -> Parser[B]` | Parse both, keep right |
| `seq_pair` | `fn(Parser[A], Parser[B]) -> Parser[(A, B)]` | Parse both, keep pair |
| `between` | `fn(Parser[A], Parser[B], Parser[C]) -> Parser[B]` | Parse between delimiters |

### Alternation

| Combinator | Signature | Description |
|------------|-----------|-------------|
| `or_else` | `fn(Parser[T], Parser[T]) -> Parser[T]` | Try first, then second |
| `choice` | `fn(List[Parser[T]]) -> Parser[T]` | Try parsers in order |
| `optional` | `fn(Parser[T]) -> Parser[Option[T]]` | Zero or one |

### Transformation

| Combinator | Signature | Description |
|------------|-----------|-------------|
| `map` | `fn(Parser[A], fn(A) -> B) -> Parser[B]` | Transform result |
| `map2` | `fn(Parser[A], Parser[B], fn(A, B) -> C) -> Parser[C]` | Combine two results |
| `map3` | `fn(Parser[A], Parser[B], Parser[C], fn(A, B, C) -> D) -> Parser[D]` | Combine three results |
| `replace` | `fn(Parser[A], B) -> Parser[B]` | Replace with constant |

### Repetition

| Combinator | Signature | Description |
|------------|-----------|-------------|
| `many` | `fn(Parser[T]) -> Parser[List[T]]` | Zero or more |
| `many1` | `fn(Parser[T]) -> Parser[List[T]]` | One or more |
| `sep_by` | `fn(Parser[T], Parser[S]) -> Parser[List[T]]` | Separated list |
| `sep_by1` | `fn(Parser[T], Parser[S]) -> Parser[List[T]]` | Non-empty separated list |
| `count` | `fn(i32, Parser[T]) -> Parser[List[T]]` | Exactly N times |

### Expression Parsing

| Combinator | Signature | Description |
|------------|-----------|-------------|
| `chainl1` | `fn(Parser[T], Parser[fn(T, T) -> T]) -> Parser[T]` | Left-associative chain |
| `chainr1` | `fn(Parser[T], Parser[fn(T, T) -> T]) -> Parser[T]` | Right-associative chain |
| `build_expr_parser` | `fn(List[PrecLevel[T]], Parser[T]) -> Parser[T]` | Precedence parser |

## Examples

### JSON Parser

The library includes a complete JSON parser in `examples/json.ki`:

```kira
import examples.json

let json: string = "{\"name\": \"Alice\", \"age\": 30}"
let result: Result[JsonValue, ParseError] = parse_json(json)

match result {
    Ok(value) => {
        let name: Option[JsonValue] = json_get(value, "name")
        // => Some(JsonString("Alice"))
    }
    Err(e) => {
        let msg: string = format_parse_error(e)
    }
}
```

### Calculator

The `examples/calc.ki` demonstrates expression parsing with operator precedence:

```kira
// Parses expressions like: 1 + 2 * 3 - 4 / 2
// Respects precedence: * and / bind tighter than + and -
```

## Error Messages

The library provides detailed error messages with position information:

```
Parse error at line 3, column 15:
  Expected: identifier, number, or '('
  Got: ')'
  Context: expression > term > factor
```

## Design Principles

1. **Pure by Default**: Core parsing is pure; effects only at IO boundaries
2. **Type-Driven**: Invalid states are unrepresentable
3. **Compositional**: Small parsers combine into complex ones
4. **Informative Errors**: Position tracking and context for debugging

## Testing

```bash
# Run all tests
kira test

# Run specific test file
kira test tests/test_combinators.ki
```

## License

MIT License - see [LICENSE](LICENSE) for details.

Copyright (c) 2026 MrPhil (Philip Ludington)
