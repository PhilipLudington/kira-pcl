# Parser Combinator Library (PCL) Design

A parser combinator library for Kira that exercises the language's features comprehensively.

## Overview

Parser combinators are higher-order functions that combine simple parsers into more complex ones. This design showcases:

- **Algebraic Data Types**: Sum types for parse results, product types for state
- **Generics**: Polymorphic parsers and combinators
- **Higher-Order Functions**: Combinators that take and return parsers
- **Purity**: Core parsing is pure; IO only at boundaries
- **Pattern Matching**: Exhaustive matching on parse results
- **Error Handling**: Rich error types with context
- **Type-Driven Design**: Invalid states unrepresentable

---

## Core Types

### Input Stream

```kira
/// Position in source input for error reporting
type Position = {
    offset: i64,      // Byte offset from start
    line: i32,        // 1-indexed line number
    column: i32       // 1-indexed column number
}

/// Input stream with position tracking
type Input = {
    source: string,   // Full source text
    pos: Position     // Current position
}

/// Create input from source string
let input_new: fn(string) -> Input = fn(source: string) -> Input {
    return Input {
        source: source,
        pos: Position { offset: 0, line: 1, column: 1 }
    }
}
```

### Parse Result (Sum Type)

```kira
/// Result of a parse attempt
type ParseResult[T] =
    | Success { value: T, remaining: Input }
    | Failure { error: ParseError }

/// Check if parse succeeded
let is_success[T]: fn(ParseResult[T]) -> bool = fn(result: ParseResult[T]) -> bool {
    match result {
        Success { value: _, remaining: _ } => { return true }
        Failure { error: _ } => { return false }
    }
}
```

### Parse Errors (Sum Type with Context)

```kira
/// Expected input description
type Expected =
    | Literal(string)
    | CharClass(string)
    | EndOfInput
    | Custom(string)

/// Parse error with full context
type ParseError = {
    position: Position,
    expected: List[Expected],
    actual: Option[char],
    context: List[string]
}

/// Create error at position
let error_at: fn(Position, Expected, Option[char]) -> ParseError =
    fn(pos: Position, exp: Expected, actual: Option[char]) -> ParseError {
        return ParseError {
            position: pos,
            expected: Cons(exp, Nil),
            actual: actual,
            context: Nil
        }
    }

/// Merge two errors (for alternation)
let merge_errors: fn(ParseError, ParseError) -> ParseError =
    fn(e1: ParseError, e2: ParseError) -> ParseError {
        // Keep error at furthest position
        if e1.position.offset > e2.position.offset {
            return e1
        }
        if e2.position.offset > e1.position.offset {
            return e2
        }
        // Same position: merge expected lists
        return ParseError {
            position: e1.position,
            expected: concat(e1.expected, e2.expected),
            actual: e1.actual,
            context: e1.context
        }
    }
```

---

## The Parser Type

### Parser as a Function Type (Higher-Order)

```kira
/// A parser is a function from input to parse result
/// This is the core abstraction of the library
type Parser[T] = fn(Input) -> ParseResult[T]
```

### Running Parsers

```kira
/// Run a parser on input string
let run[T]: fn(Parser[T], string) -> ParseResult[T] =
    fn(parser: Parser[T], source: string) -> ParseResult[T] {
        let input: Input = input_new(source)
        return parser(input)
    }

/// Run parser and extract value or error
let parse[T]: fn(Parser[T], string) -> Result[T, ParseError] =
    fn(parser: Parser[T], source: string) -> Result[T, ParseError] {
        match run(parser, source) {
            Success { value: v, remaining: _ } => { return Ok(v) }
            Failure { error: e } => { return Err(e) }
        }
    }

/// Run parser requiring complete consumption
let parse_complete[T]: fn(Parser[T], string) -> Result[T, ParseError] =
    fn(parser: Parser[T], source: string) -> Result[T, ParseError] {
        let combined: Parser[T] = seq_left(parser, eof())
        return parse(combined, source)
    }
```

---

## Primitive Parsers

### Character Parsers

```kira
/// Parse any single character
let any_char: fn() -> Parser[char] = fn() -> Parser[char] {
    return fn(input: Input) -> ParseResult[char] {
        let remaining: string = substring(input.source, input.pos.offset)
        match string_head(remaining) {
            None => {
                return Failure {
                    error: error_at(input.pos, CharClass("any character"), None)
                }
            }
            Some(c) => {
                let new_pos: Position = advance_position(input.pos, c)
                return Success {
                    value: c,
                    remaining: Input { source: input.source, pos: new_pos }
                }
            }
        }
    }
}

/// Parse a specific character
let char_p: fn(char) -> Parser[char] = fn(expected: char) -> Parser[char] {
    return fn(input: Input) -> ParseResult[char] {
        let remaining: string = substring(input.source, input.pos.offset)
        match string_head(remaining) {
            None => {
                return Failure {
                    error: error_at(input.pos, Literal(char_to_string(expected)), None)
                }
            }
            Some(c) => {
                if c == expected {
                    let new_pos: Position = advance_position(input.pos, c)
                    return Success {
                        value: c,
                        remaining: Input { source: input.source, pos: new_pos }
                    }
                }
                return Failure {
                    error: error_at(input.pos, Literal(char_to_string(expected)), Some(c))
                }
            }
        }
    }
}

/// Parse character satisfying predicate
let satisfy: fn(fn(char) -> bool, string) -> Parser[char] =
    fn(pred: fn(char) -> bool, description: string) -> Parser[char] {
        return fn(input: Input) -> ParseResult[char] {
            let remaining: string = substring(input.source, input.pos.offset)
            match string_head(remaining) {
                None => {
                    return Failure {
                        error: error_at(input.pos, CharClass(description), None)
                    }
                }
                Some(c) => {
                    if pred(c) {
                        let new_pos: Position = advance_position(input.pos, c)
                        return Success {
                            value: c,
                            remaining: Input { source: input.source, pos: new_pos }
                        }
                    }
                    return Failure {
                        error: error_at(input.pos, CharClass(description), Some(c))
                    }
                }
            }
        }
    }
```

### Character Class Parsers

```kira
/// Parse a digit character
let digit: fn() -> Parser[char] = fn() -> Parser[char] {
    return satisfy(is_digit, "digit")
}

/// Parse a letter character
let letter: fn() -> Parser[char] = fn() -> Parser[char] {
    return satisfy(is_alpha, "letter")
}

/// Parse alphanumeric character
let alphanumeric: fn() -> Parser[char] = fn() -> Parser[char] {
    return satisfy(is_alphanumeric, "alphanumeric")
}

/// Parse whitespace character
let whitespace: fn() -> Parser[char] = fn() -> Parser[char] {
    return satisfy(is_whitespace, "whitespace")
}

/// Parse one of given characters
let one_of: fn(string) -> Parser[char] = fn(chars: string) -> Parser[char] {
    let pred: fn(char) -> bool = fn(c: char) -> bool {
        return string_contains_char(chars, c)
    }
    return satisfy(pred, "one of '{chars}'")
}

/// Parse none of given characters
let none_of: fn(string) -> Parser[char] = fn(chars: string) -> Parser[char] {
    let pred: fn(char) -> bool = fn(c: char) -> bool {
        return not string_contains_char(chars, c)
    }
    return satisfy(pred, "none of '{chars}'")
}
```

### String Parsers

```kira
/// Parse exact string
let string_p: fn(string) -> Parser[string] = fn(expected: string) -> Parser[string] {
    return fn(input: Input) -> ParseResult[string] {
        let remaining: string = substring(input.source, input.pos.offset)
        if starts_with(remaining, expected) {
            let new_pos: Position = advance_position_by_string(input.pos, expected)
            return Success {
                value: expected,
                remaining: Input { source: input.source, pos: new_pos }
            }
        }
        let actual: Option[char] = string_head(remaining)
        return Failure {
            error: error_at(input.pos, Literal(expected), actual)
        }
    }
}

/// Parse end of input
let eof: fn() -> Parser[void] = fn() -> Parser[void] {
    return fn(input: Input) -> ParseResult[void] {
        let remaining: string = substring(input.source, input.pos.offset)
        if string_is_empty(remaining) {
            return Success { value: void, remaining: input }
        }
        let actual: Option[char] = string_head(remaining)
        return Failure {
            error: error_at(input.pos, EndOfInput, actual)
        }
    }
}
```

### Pure Parsers

```kira
/// Parser that always succeeds with given value (monadic return)
let pure[T]: fn(T) -> Parser[T] = fn(value: T) -> Parser[T] {
    return fn(input: Input) -> ParseResult[T] {
        return Success { value: value, remaining: input }
    }
}

/// Parser that always fails with given message
let fail[T]: fn(string) -> Parser[T] = fn(message: string) -> Parser[T] {
    return fn(input: Input) -> ParseResult[T] {
        return Failure {
            error: error_at(input.pos, Custom(message), None)
        }
    }
}
```

---

## Combinators

### Sequencing (Monadic Bind)

```kira
/// Sequence two parsers, combining results (monadic bind / flatMap)
let and_then[A, B]: fn(Parser[A], fn(A) -> Parser[B]) -> Parser[B] =
    fn(pa: Parser[A], f: fn(A) -> Parser[B]) -> Parser[B] {
        return fn(input: Input) -> ParseResult[B] {
            match pa(input) {
                Failure { error: e } => { return Failure { error: e } }
                Success { value: a, remaining: rest } => {
                    let pb: Parser[B] = f(a)
                    return pb(rest)
                }
            }
        }
    }

/// Sequence two parsers, keep left result
let seq_left[A, B]: fn(Parser[A], Parser[B]) -> Parser[A] =
    fn(pa: Parser[A], pb: Parser[B]) -> Parser[A] {
        return and_then[A, A](pa, fn(a: A) -> Parser[A] {
            return and_then[B, A](pb, fn(_: B) -> Parser[A] {
                return pure(a)
            })
        })
    }

/// Sequence two parsers, keep right result
let seq_right[A, B]: fn(Parser[A], Parser[B]) -> Parser[B] =
    fn(pa: Parser[A], pb: Parser[B]) -> Parser[B] {
        return and_then[A, B](pa, fn(_: A) -> Parser[B] {
            return pb
        })
    }

/// Sequence two parsers, keep both results as pair
let seq_pair[A, B]: fn(Parser[A], Parser[B]) -> Parser[(A, B)] =
    fn(pa: Parser[A], pb: Parser[B]) -> Parser[(A, B)] {
        return and_then[A, (A, B)](pa, fn(a: A) -> Parser[(A, B)] {
            return and_then[B, (A, B)](pb, fn(b: B) -> Parser[(A, B)] {
                return pure((a, b))
            })
        })
    }

/// Sequence three parsers, keep middle result
let between[A, B, C]: fn(Parser[A], Parser[B], Parser[C]) -> Parser[B] =
    fn(open: Parser[A], content: Parser[B], close: Parser[C]) -> Parser[B] {
        return seq_left(seq_right(open, content), close)
    }
```

### Alternation

```kira
/// Try first parser, if it fails try second (with error merging)
let or_else[T]: fn(Parser[T], Parser[T]) -> Parser[T] =
    fn(p1: Parser[T], p2: Parser[T]) -> Parser[T] {
        return fn(input: Input) -> ParseResult[T] {
            match p1(input) {
                Success { value: v, remaining: r } => {
                    return Success { value: v, remaining: r }
                }
                Failure { error: e1 } => {
                    match p2(input) {
                        Success { value: v, remaining: r } => {
                            return Success { value: v, remaining: r }
                        }
                        Failure { error: e2 } => {
                            return Failure { error: merge_errors(e1, e2) }
                        }
                    }
                }
            }
        }
    }

/// Try parsers in order, return first success
let choice[T]: fn(List[Parser[T]]) -> Parser[T] =
    fn(parsers: List[Parser[T]]) -> Parser[T] {
        match parsers {
            Nil => { return fail("no alternatives") }
            Cons(p, Nil) => { return p }
            Cons(p, rest) => { return or_else(p, choice(rest)) }
        }
    }

/// Optional parser - returns Option[T]
let optional[T]: fn(Parser[T]) -> Parser[Option[T]] =
    fn(parser: Parser[T]) -> Parser[Option[T]] {
        return or_else(
            map[T, Option[T]](parser, fn(v: T) -> Option[T] { return Some(v) }),
            pure(None)
        )
    }
```

### Transformation (Functor Map)

```kira
/// Transform parser result (functor map)
let map[A, B]: fn(Parser[A], fn(A) -> B) -> Parser[B] =
    fn(parser: Parser[A], f: fn(A) -> B) -> Parser[B] {
        return fn(input: Input) -> ParseResult[B] {
            match parser(input) {
                Failure { error: e } => { return Failure { error: e } }
                Success { value: a, remaining: rest } => {
                    return Success { value: f(a), remaining: rest }
                }
            }
        }
    }

/// Transform with two parsers (applicative lift)
let map2[A, B, C]: fn(Parser[A], Parser[B], fn(A, B) -> C) -> Parser[C] =
    fn(pa: Parser[A], pb: Parser[B], f: fn(A, B) -> C) -> Parser[C] {
        return and_then[A, C](pa, fn(a: A) -> Parser[C] {
            return map[B, C](pb, fn(b: B) -> C {
                return f(a, b)
            })
        })
    }

/// Transform with three parsers
let map3[A, B, C, D]: fn(Parser[A], Parser[B], Parser[C], fn(A, B, C) -> D) -> Parser[D] =
    fn(pa: Parser[A], pb: Parser[B], pc: Parser[C], f: fn(A, B, C) -> D) -> Parser[D] {
        return and_then[A, D](pa, fn(a: A) -> Parser[D] {
            return and_then[B, D](pb, fn(b: B) -> Parser[D] {
                return map[C, D](pc, fn(c: C) -> D {
                    return f(a, b, c)
                })
            })
        })
    }

/// Replace successful result with constant
let replace[A, B]: fn(Parser[A], B) -> Parser[B] =
    fn(parser: Parser[A], value: B) -> Parser[B] {
        return map[A, B](parser, fn(_: A) -> B { return value })
    }
```

### Repetition

```kira
/// Parse zero or more occurrences
let many[T]: fn(Parser[T]) -> Parser[List[T]] =
    fn(parser: Parser[T]) -> Parser[List[T]] {
        return fn(input: Input) -> ParseResult[List[T]] {
            var results: List[T] = Nil
            var current: Input = input
            loop {
                match parser(current) {
                    Failure { error: _ } => {
                        return Success {
                            value: reverse(results),
                            remaining: current
                        }
                    }
                    Success { value: v, remaining: rest } => {
                        results = Cons(v, results)
                        current = rest
                    }
                }
            }
        }
    }

/// Parse one or more occurrences
let many1[T]: fn(Parser[T]) -> Parser[List[T]] =
    fn(parser: Parser[T]) -> Parser[List[T]] {
        return and_then[T, List[T]](parser, fn(first: T) -> Parser[List[T]] {
            return map[List[T], List[T]](many(parser), fn(rest: List[T]) -> List[T] {
                return Cons(first, rest)
            })
        })
    }

/// Parse exactly n occurrences
let count[T]: fn(i32, Parser[T]) -> Parser[List[T]] =
    fn(n: i32, parser: Parser[T]) -> Parser[List[T]] {
        if n <= 0 {
            return pure(Nil)
        }
        return and_then[T, List[T]](parser, fn(x: T) -> Parser[List[T]] {
            return map[List[T], List[T]](count(n - 1, parser), fn(xs: List[T]) -> List[T] {
                return Cons(x, xs)
            })
        })
    }

/// Parse separated list (zero or more)
let sep_by[T, S]: fn(Parser[T], Parser[S]) -> Parser[List[T]] =
    fn(item: Parser[T], sep: Parser[S]) -> Parser[List[T]] {
        return or_else(sep_by1(item, sep), pure(Nil))
    }

/// Parse separated list (one or more)
let sep_by1[T, S]: fn(Parser[T], Parser[S]) -> Parser[List[T]] =
    fn(item: Parser[T], sep: Parser[S]) -> Parser[List[T]] {
        return and_then[T, List[T]](item, fn(first: T) -> Parser[List[T]] {
            let rest_parser: Parser[List[T]] = many(seq_right(sep, item))
            return map[List[T], List[T]](rest_parser, fn(rest: List[T]) -> List[T] {
                return Cons(first, rest)
            })
        })
    }

/// Parse with ending separator
let end_by[T, S]: fn(Parser[T], Parser[S]) -> Parser[List[T]] =
    fn(item: Parser[T], sep: Parser[S]) -> Parser[List[T]] {
        return many(seq_left(item, sep))
    }
```

### Lookahead and Backtracking

```kira
/// Lookahead - succeed if parser succeeds but don't consume input
let lookahead[T]: fn(Parser[T]) -> Parser[T] =
    fn(parser: Parser[T]) -> Parser[T] {
        return fn(input: Input) -> ParseResult[T] {
            match parser(input) {
                Failure { error: e } => { return Failure { error: e } }
                Success { value: v, remaining: _ } => {
                    return Success { value: v, remaining: input }
                }
            }
        }
    }

/// Negative lookahead - succeed if parser fails
let not_followed_by[T]: fn(Parser[T]) -> Parser[void] =
    fn(parser: Parser[T]) -> Parser[void] {
        return fn(input: Input) -> ParseResult[void] {
            match parser(input) {
                Success { value: _, remaining: _ } => {
                    return Failure {
                        error: error_at(input.pos, Custom("unexpected match"), None)
                    }
                }
                Failure { error: _ } => {
                    return Success { value: void, remaining: input }
                }
            }
        }
    }

/// Try parser with full backtracking on failure
let try_p[T]: fn(Parser[T]) -> Parser[T] =
    fn(parser: Parser[T]) -> Parser[T] {
        return fn(input: Input) -> ParseResult[T] {
            match parser(input) {
                Success { value: v, remaining: r } => {
                    return Success { value: v, remaining: r }
                }
                Failure { error: e } => {
                    // Reset position in error to input position
                    return Failure {
                        error: ParseError {
                            position: input.pos,
                            expected: e.expected,
                            actual: e.actual,
                            context: e.context
                        }
                    }
                }
            }
        }
    }
```

### Error Enhancement

```kira
/// Add context label to error messages
let label[T]: fn(Parser[T], string) -> Parser[T] =
    fn(parser: Parser[T], name: string) -> Parser[T] {
        return fn(input: Input) -> ParseResult[T] {
            match parser(input) {
                Success { value: v, remaining: r } => {
                    return Success { value: v, remaining: r }
                }
                Failure { error: e } => {
                    return Failure {
                        error: ParseError {
                            position: e.position,
                            expected: Cons(Custom(name), Nil),
                            actual: e.actual,
                            context: e.context
                        }
                    }
                }
            }
        }
    }

/// Add context to error for debugging
let context[T]: fn(Parser[T], string) -> Parser[T] =
    fn(parser: Parser[T], ctx: string) -> Parser[T] {
        return fn(input: Input) -> ParseResult[T] {
            match parser(input) {
                Success { value: v, remaining: r } => {
                    return Success { value: v, remaining: r }
                }
                Failure { error: e } => {
                    return Failure {
                        error: ParseError {
                            position: e.position,
                            expected: e.expected,
                            actual: e.actual,
                            context: Cons(ctx, e.context)
                        }
                    }
                }
            }
        }
    }
```

---

## Recursive Parsers

### Lazy Evaluation for Recursion

```kira
/// Delay parser construction for recursion
type LazyParser[T] = {
    thunk: fn() -> Parser[T]
}

/// Create lazy parser
let lazy[T]: fn(fn() -> Parser[T]) -> Parser[T] =
    fn(thunk: fn() -> Parser[T]) -> Parser[T] {
        return fn(input: Input) -> ParseResult[T] {
            let parser: Parser[T] = thunk()
            return parser(input)
        }
    }

/// Fix-point combinator for mutually recursive parsers
let fix[T]: fn(fn(Parser[T]) -> Parser[T]) -> Parser[T] =
    fn(f: fn(Parser[T]) -> Parser[T]) -> Parser[T] {
        return lazy(fn() -> Parser[T] {
            return f(fix(f))
        })
    }
```

---

## Expression Parsing

### Operator Precedence

```kira
/// Operator associativity
type Assoc =
    | AssocLeft
    | AssocRight
    | AssocNone

/// Operator definition
type Operator[T] = {
    parser: Parser[fn(T, T) -> T],
    assoc: Assoc
}

/// Precedence level (list of operators at same level)
type PrecLevel[T] = List[Operator[T]]

/// Build expression parser from precedence table
let build_expr_parser[T]: fn(List[PrecLevel[T]], Parser[T]) -> Parser[T] =
    fn(table: List[PrecLevel[T]], term: Parser[T]) -> Parser[T] {
        return fold[PrecLevel[T], Parser[T]](
            table,
            term,
            fn(inner: Parser[T], level: PrecLevel[T]) -> Parser[T] {
                return add_operators(inner, level)
            }
        )
    }

/// Add operators at one precedence level
let add_operators[T]: fn(Parser[T], PrecLevel[T]) -> Parser[T] =
    fn(inner: Parser[T], ops: PrecLevel[T]) -> Parser[T] {
        let op_parser: Parser[fn(T, T) -> T] = choice(
            list_map[Operator[T], Parser[fn(T, T) -> T]](
                ops,
                fn(op: Operator[T]) -> Parser[fn(T, T) -> T] { return op.parser }
            )
        )
        // Simplified: assume all left-associative for this example
        return chainl1(inner, op_parser)
    }

/// Left-associative chain
let chainl1[T]: fn(Parser[T], Parser[fn(T, T) -> T]) -> Parser[T] =
    fn(term: Parser[T], op: Parser[fn(T, T) -> T]) -> Parser[T] {
        return and_then[T, T](term, fn(first: T) -> Parser[T] {
            return chainl1_rest(first, term, op)
        })
    }

let chainl1_rest[T]: fn(T, Parser[T], Parser[fn(T, T) -> T]) -> Parser[T] =
    fn(acc: T, term: Parser[T], op: Parser[fn(T, T) -> T]) -> Parser[T] {
        let more: Parser[T] = and_then[fn(T, T) -> T, T](op, fn(f: fn(T, T) -> T) -> Parser[T] {
            return and_then[T, T](term, fn(next: T) -> Parser[T] {
                let new_acc: T = f(acc, next)
                return chainl1_rest(new_acc, term, op)
            })
        })
        return or_else(more, pure(acc))
    }

/// Right-associative chain
let chainr1[T]: fn(Parser[T], Parser[fn(T, T) -> T]) -> Parser[T] =
    fn(term: Parser[T], op: Parser[fn(T, T) -> T]) -> Parser[T] {
        return and_then[T, T](term, fn(first: T) -> Parser[T] {
            return or_else(
                and_then[fn(T, T) -> T, T](op, fn(f: fn(T, T) -> T) -> Parser[T] {
                    return map[T, T](chainr1(term, op), fn(rest: T) -> T {
                        return f(first, rest)
                    })
                }),
                pure(first)
            )
        })
    }
```

---

## Lexer Helpers

### Token Combinators

```kira
/// Skip whitespace
let skip_ws: fn() -> Parser[void] = fn() -> Parser[void] {
    return replace(many(whitespace()), void)
}

/// Parse with trailing whitespace
let lexeme[T]: fn(Parser[T]) -> Parser[T] = fn(parser: Parser[T]) -> Parser[T] {
    return seq_left(parser, skip_ws())
}

/// Parse keyword (not followed by alphanumeric)
let keyword: fn(string) -> Parser[string] = fn(kw: string) -> Parser[string] {
    return lexeme(seq_left(string_p(kw), not_followed_by(alphanumeric())))
}

/// Parse symbol with whitespace
let symbol: fn(string) -> Parser[string] = fn(sym: string) -> Parser[string] {
    return lexeme(string_p(sym))
}

/// Parse identifier (letter followed by alphanumerics)
let identifier: fn() -> Parser[string] = fn() -> Parser[string] {
    return lexeme(
        map2[char, List[char], string](
            letter(),
            many(alphanumeric()),
            fn(first: char, rest: List[char]) -> string {
                return chars_to_string(Cons(first, rest))
            }
        )
    )
}

/// Parse integer literal
let integer: fn() -> Parser[i64] = fn() -> Parser[i64] {
    let digits: Parser[string] = map[List[char], string](
        many1(digit()),
        fn(chars: List[char]) -> string { return chars_to_string(chars) }
    )
    return lexeme(map[string, i64](digits, string_to_i64))
}

/// Parse string literal
let string_literal: fn() -> Parser[string] = fn() -> Parser[string] {
    let quote: Parser[char] = char_p('"')
    let escaped: Parser[char] = seq_right(
        char_p('\\'),
        choice([
            replace(char_p('n'), '\n'),
            replace(char_p('t'), '\t'),
            replace(char_p('\\'), '\\'),
            replace(char_p('"'), '"')
        ])
    )
    let normal: Parser[char] = satisfy(
        fn(c: char) -> bool { return c != '"' and c != '\\' },
        "string character"
    )
    let char_content: Parser[char] = or_else(escaped, normal)
    return lexeme(between(
        quote,
        map[List[char], string](many(char_content), chars_to_string),
        quote
    ))
}
```

---

## Example: JSON Parser

### JSON AST (Sum Type)

```kira
/// JSON value representation
type JsonValue =
    | JsonNull
    | JsonBool(bool)
    | JsonNumber(f64)
    | JsonString(string)
    | JsonArray(List[JsonValue])
    | JsonObject(List[(string, JsonValue)])
```

### JSON Parser Implementation

```kira
module json

/// Parse JSON null
let json_null: fn() -> Parser[JsonValue] = fn() -> Parser[JsonValue] {
    return replace(keyword("null"), JsonNull)
}

/// Parse JSON boolean
let json_bool: fn() -> Parser[JsonValue] = fn() -> Parser[JsonValue] {
    return or_else(
        replace(keyword("true"), JsonBool(true)),
        replace(keyword("false"), JsonBool(false))
    )
}

/// Parse JSON number
let json_number: fn() -> Parser[JsonValue] = fn() -> Parser[JsonValue] {
    // Simplified: just integers for this example
    return map[i64, JsonValue](integer(), fn(n: i64) -> JsonValue {
        return JsonNumber(i64_to_f64(n))
    })
}

/// Parse JSON string
let json_string: fn() -> Parser[JsonValue] = fn() -> Parser[JsonValue] {
    return map[string, JsonValue](string_literal(), fn(s: string) -> JsonValue {
        return JsonString(s)
    })
}

/// Parse JSON array (recursive)
let json_array: fn() -> Parser[JsonValue] = fn() -> Parser[JsonValue] {
    return map[List[JsonValue], JsonValue](
        between(
            symbol("["),
            sep_by(lazy(json_value), symbol(",")),
            symbol("]")
        ),
        fn(items: List[JsonValue]) -> JsonValue { return JsonArray(items) }
    )
}

/// Parse JSON object (recursive)
let json_object: fn() -> Parser[JsonValue] = fn() -> Parser[JsonValue] {
    let pair: Parser[(string, JsonValue)] = map2[string, JsonValue, (string, JsonValue)](
        seq_left(string_literal(), symbol(":")),
        lazy(json_value),
        fn(key: string, value: JsonValue) -> (string, JsonValue) {
            return (key, value)
        }
    )
    return map[List[(string, JsonValue)], JsonValue](
        between(
            symbol("{"),
            sep_by(pair, symbol(",")),
            symbol("}")
        ),
        fn(pairs: List[(string, JsonValue)]) -> JsonValue { return JsonObject(pairs) }
    )
}

/// Parse any JSON value
let json_value: fn() -> Parser[JsonValue] = fn() -> Parser[JsonValue] {
    return label(
        choice([
            json_null(),
            json_bool(),
            json_number(),
            json_string(),
            json_array(),
            json_object()
        ]),
        "JSON value"
    )
}

/// Parse complete JSON document
pub let parse_json: fn(string) -> Result[JsonValue, ParseError] =
    fn(source: string) -> Result[JsonValue, ParseError] {
        let parser: Parser[JsonValue] = seq_right(skip_ws(), json_value())
        return parse_complete(parser, source)
    }
```

---

## Example: Arithmetic Expression Parser

### Expression AST

```kira
/// Arithmetic expression
type Expr =
    | Literal(i64)
    | Add(Expr, Expr)
    | Sub(Expr, Expr)
    | Mul(Expr, Expr)
    | Div(Expr, Expr)
    | Neg(Expr)
    | Parens(Expr)
```

### Expression Parser

```kira
module expr

/// Parse integer literal
let literal: fn() -> Parser[Expr] = fn() -> Parser[Expr] {
    return map[i64, Expr](integer(), fn(n: i64) -> Expr {
        return Literal(n)
    })
}

/// Parse parenthesized expression
let parens: fn() -> Parser[Expr] = fn() -> Parser[Expr] {
    return map[Expr, Expr](
        between(symbol("("), lazy(expr), symbol(")")),
        fn(e: Expr) -> Expr { return Parens(e) }
    )
}

/// Parse unary negation
let negation: fn() -> Parser[Expr] = fn() -> Parser[Expr] {
    return map[Expr, Expr](
        seq_right(symbol("-"), lazy(fn() -> Parser[Expr] { return factor() })),
        fn(e: Expr) -> Expr { return Neg(e) }
    )
}

/// Parse factor (atom or negation)
let factor: fn() -> Parser[Expr] = fn() -> Parser[Expr] {
    return choice([negation(), parens(), literal()])
}

/// Parse term with * and /
let term: fn() -> Parser[Expr] = fn() -> Parser[Expr] {
    let mul_op: Parser[fn(Expr, Expr) -> Expr] = replace(
        symbol("*"),
        fn(a: Expr, b: Expr) -> Expr { return Mul(a, b) }
    )
    let div_op: Parser[fn(Expr, Expr) -> Expr] = replace(
        symbol("/"),
        fn(a: Expr, b: Expr) -> Expr { return Div(a, b) }
    )
    return chainl1(factor(), or_else(mul_op, div_op))
}

/// Parse expression with + and -
let expr: fn() -> Parser[Expr] = fn() -> Parser[Expr] {
    let add_op: Parser[fn(Expr, Expr) -> Expr] = replace(
        symbol("+"),
        fn(a: Expr, b: Expr) -> Expr { return Add(a, b) }
    )
    let sub_op: Parser[fn(Expr, Expr) -> Expr] = replace(
        symbol("-"),
        fn(a: Expr, b: Expr) -> Expr { return Sub(a, b) }
    )
    return chainl1(term(), or_else(add_op, sub_op))
}

/// Parse and evaluate expression
pub let parse_expr: fn(string) -> Result[Expr, ParseError] =
    fn(source: string) -> Result[Expr, ParseError] {
        let parser: Parser[Expr] = seq_right(skip_ws(), expr())
        return parse_complete(parser, source)
    }

/// Evaluate expression
pub let eval: fn(Expr) -> i64 = fn(e: Expr) -> i64 {
    match e {
        Literal(n) => { return n }
        Add(a, b) => { return eval(a) + eval(b) }
        Sub(a, b) => { return eval(a) - eval(b) }
        Mul(a, b) => { return eval(a) * eval(b) }
        Div(a, b) => { return eval(a) / eval(b) }
        Neg(a) => { return -eval(a) }
        Parens(a) => { return eval(a) }
    }
}
```

---

## Effect Integration

### Parsing from Files

```kira
module pcl.io

/// Parse file contents
effect fn parse_file[T](path: string, parser: Parser[T]) -> IO[Result[T, ParseFileError]] {
    let content: Result[string, IoError] = std.fs.read_file(path)
    match content {
        Err(io_err) => {
            return Err(ParseFileError.IoError(io_err))
        }
        Ok(source) => {
            match parse_complete(parser, source) {
                Ok(value) => { return Ok(value) }
                Err(parse_err) => {
                    return Err(ParseFileError.ParseError {
                        path: path,
                        error: parse_err
                    })
                }
            }
        }
    }
}

/// Error type for file parsing
type ParseFileError =
    | IoError(IoError)
    | ParseError { path: string, error: ParseError }

/// Format error for display
effect fn format_error(error: ParseFileError) -> IO[string] {
    match error {
        IoError(e) => {
            return "IO Error: {e}"
        }
        ParseError { path: p, error: e } => {
            return "Parse error in {p} at line {e.position.line}, column {e.position.column}"
        }
    }
}
```

---

## Module Structure

```
pcl/
├── src/
│   ├── pcl.ki              # Main module, re-exports
│   ├── types.ki            # Core types (Input, ParseResult, etc.)
│   ├── primitives.ki       # Primitive parsers
│   ├── combinators.ki      # All combinators
│   ├── expr.ki             # Expression parsing utilities
│   ├── lexer.ki            # Lexer helpers
│   └── io.ki               # Effect-ful IO operations
├── examples/
│   ├── json.ki             # JSON parser example
│   ├── calc.ki             # Calculator example
│   └── lisp.ki             # S-expression parser
└── tests/
    ├── test_primitives.ki
    ├── test_combinators.ki
    ├── test_json.ki
    └── test_expr.ki
```

---

## Kira Features Exercised

| Feature | Where Used |
|---------|------------|
| **Sum Types** | `ParseResult`, `Expected`, `JsonValue`, `Expr`, `Assoc` |
| **Product Types** | `Position`, `Input`, `ParseError`, `Operator` |
| **Generics** | All parser types and combinators |
| **Higher-Order Functions** | Core `Parser[T]` type, all combinators |
| **Closures** | Every parser construction |
| **Pattern Matching** | All result handling, AST evaluation |
| **Explicit Types** | All function signatures |
| **Explicit Return** | Every function body |
| **Pure Functions** | Core parsing logic |
| **Effect Functions** | File I/O operations |
| **Result/Option** | Error handling |
| **Recursion** | `many`, recursive parsers |
| **Modules** | `json`, `expr`, `pcl.io` |
| **Type Aliases** | `Parser[T]` |
| **Documentation** | Function docs |

---

## Design Principles

1. **Composability**: Small parsers combine into complex ones
2. **Purity**: Parsing is pure; effects only at boundaries
3. **Type Safety**: Parser types prevent invalid compositions
4. **Error Quality**: Position tracking and context for debugging
5. **Laziness**: Recursive parsers via thunks
6. **Extensibility**: Easy to add new combinators

---

## API Summary

### Core Types
- `Parser[T]` - A parser producing values of type T
- `ParseResult[T]` - Success or Failure
- `Input` - Input stream with position

### Running Parsers
- `run`, `parse`, `parse_complete`

### Primitives
- `any_char`, `char_p`, `satisfy`, `string_p`, `eof`, `pure`, `fail`

### Character Classes
- `digit`, `letter`, `alphanumeric`, `whitespace`, `one_of`, `none_of`

### Combinators
- `and_then`, `map`, `map2`, `map3`
- `or_else`, `choice`, `optional`
- `seq_left`, `seq_right`, `seq_pair`, `between`
- `many`, `many1`, `count`, `sep_by`, `sep_by1`, `end_by`
- `lookahead`, `not_followed_by`, `try_p`
- `label`, `context`, `lazy`, `fix`

### Expression Parsing
- `chainl1`, `chainr1`, `build_expr_parser`

### Lexer Helpers
- `skip_ws`, `lexeme`, `keyword`, `symbol`, `identifier`, `integer`, `string_literal`
