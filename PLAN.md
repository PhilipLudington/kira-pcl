# Parser Combinator Library - Implementation Plan

## Project Overview

Implementation of a Parser Combinator Library (PCL) for Kira, following the design in `DESIGN.md`.

**Goal**: Exercise Kira language features comprehensively through a practical, composable parsing library.

---

## Phase 1: Foundation - Core Types and Infrastructure

### Task 1.1: Project Structure Setup
**File**: Directory structure
**Status**: [x] Complete

Create the project directory structure:
```
src/
  pcl.ki
  types.ki
  primitives.ki
  combinators.ki
  expr.ki
  lexer.ki
  io.ki
examples/
  json.ki
  calc.ki
  lisp.ki
tests/
  test_types.ki
  test_primitives.ki
  test_combinators.ki
  test_lexer.ki
  test_expr.ki
  test_json.ki
```

### Task 1.2: Core Types Module
**File**: `src/types.ki`
**Status**: [x] Complete
**Dependencies**: None

Implement foundational types:

```
Types to implement:
- Position (product type)
- Input (product type)
- Expected (sum type: Literal, CharClass, EndOfInput, Custom)
- ParseError (product type with context)
- ParseResult[T] (sum type: Success, Failure)
- Parser[T] (type alias for fn(Input) -> ParseResult[T])

Helper functions:
- position_new: fn(i64, i32, i32) -> Position
- input_new: fn(string) -> Input
- error_at: fn(Position, Expected, Option[char]) -> ParseError
- merge_errors: fn(ParseError, ParseError) -> ParseError
- is_success[T]: fn(ParseResult[T]) -> bool
- is_failure[T]: fn(ParseResult[T]) -> bool
```

**Kira Features Exercised**:
- Sum types with multiple variants
- Product types with named fields
- Generic types
- Type aliases
- Pattern matching

### Task 1.3: String Utility Functions
**File**: `src/types.ki` (or separate `src/string_utils.ki`)
**Status**: [x] Complete
**Dependencies**: Task 1.2

Implement string manipulation helpers needed by parsers:

```
Functions:
- substring: fn(string, i64) -> string
- string_head: fn(string) -> Option[char]
- string_is_empty: fn(string) -> bool
- starts_with: fn(string, string) -> bool
- string_contains_char: fn(string, char) -> bool
- char_to_string: fn(char) -> string
- chars_to_string: fn(List[char]) -> string
- string_to_i64: fn(string) -> i64
- i64_to_f64: fn(i64) -> f64

Position helpers:
- advance_position: fn(Position, char) -> Position
- advance_position_by_string: fn(Position, string) -> Position
```

### Task 1.4: List Utility Functions
**File**: `src/types.ki`
**Status**: [x] Complete
**Dependencies**: Task 1.2

Implement list operations:

```
Functions:
- reverse[T]: fn(List[T]) -> List[T]
- concat[T]: fn(List[T], List[T]) -> List[T]
- length[T]: fn(List[T]) -> i32
- list_map[A, B]: fn(List[A], fn(A) -> B) -> List[B]
- fold[A, B]: fn(List[A], B, fn(B, A) -> B) -> B
```

### Task 1.5: Types Module Tests
**File**: `tests/test_types.ki`
**Status**: [x] Complete
**Dependencies**: Tasks 1.2, 1.3, 1.4

```
Tests:
- test_position_new
- test_input_new
- test_advance_position_newline
- test_advance_position_regular_char
- test_error_at_creates_error
- test_merge_errors_takes_furthest
- test_merge_errors_same_position_merges
- test_parse_result_is_success
- test_parse_result_is_failure
- test_reverse_list
- test_concat_lists
```

---

## Phase 2: Primitive Parsers

### Task 2.1: Basic Character Parsers
**File**: `src/primitives.ki`
**Status**: [x] Complete
**Dependencies**: Phase 1 complete

Implement fundamental parsers:

```
Parsers:
- any_char: fn() -> Parser[char]
- char_p: fn(char) -> Parser[char]
- satisfy: fn(fn(char) -> bool, string) -> Parser[char]
```

**Kira Features Exercised**:
- Higher-order functions (parsers return functions)
- Closures capturing parameters
- Pattern matching on Option

### Task 2.2: Character Class Parsers
**File**: `src/primitives.ki`
**Status**: [x] Complete
**Dependencies**: Task 2.1

Implement character class parsers:

```
Character predicates:
- is_digit: fn(char) -> bool
- is_alpha: fn(char) -> bool
- is_alphanumeric: fn(char) -> bool
- is_whitespace: fn(char) -> bool

Parsers:
- digit: fn() -> Parser[char]
- letter: fn() -> Parser[char]
- alphanumeric: fn() -> Parser[char]
- whitespace: fn() -> Parser[char]
- one_of: fn(string) -> Parser[char]
- none_of: fn(string) -> Parser[char]
```

### Task 2.3: String and Terminal Parsers
**File**: `src/primitives.ki`
**Status**: [x] Complete
**Dependencies**: Task 2.1

Implement string-level parsers:

```
Parsers:
- string_p: fn(string) -> Parser[string]
- eof: fn() -> Parser[void]
- pure[T]: fn(T) -> Parser[T]
- fail[T]: fn(string) -> Parser[T]
```

### Task 2.4: Parser Execution Functions
**File**: `src/primitives.ki`
**Status**: [x] Complete
**Dependencies**: Tasks 2.1-2.3

Implement parser runners:

```
Functions:
- run[T]: fn(Parser[T], string) -> ParseResult[T]
- parse[T]: fn(Parser[T], string) -> Result[T, ParseError]
- parse_complete[T]: fn(Parser[T], string) -> Result[T, ParseError]
```

Note: `parse_complete` depends on `seq_left` from combinators - implement a basic version or defer.

### Task 2.5: Primitives Module Tests
**File**: `tests/test_primitives.ki`
**Status**: [x] Complete
**Dependencies**: Tasks 2.1-2.4

```
Tests:
- test_any_char_success
- test_any_char_empty_input
- test_char_p_match
- test_char_p_no_match
- test_satisfy_digit
- test_satisfy_custom_predicate
- test_digit_parses_digits
- test_letter_parses_letters
- test_one_of_matches
- test_none_of_excludes
- test_string_p_exact_match
- test_string_p_partial_no_match
- test_eof_at_end
- test_eof_not_at_end
- test_pure_always_succeeds
- test_fail_always_fails
- test_run_returns_parse_result
- test_parse_returns_result
```

---

## Phase 3: Core Combinators

### Task 3.1: Sequencing Combinators
**File**: `src/combinators.ki`
**Status**: [x] Complete
**Dependencies**: Phase 2 complete

Implement monadic sequencing:

```
Combinators:
- and_then[A, B]: fn(Parser[A], fn(A) -> Parser[B]) -> Parser[B]
- seq_left[A, B]: fn(Parser[A], Parser[B]) -> Parser[A]
- seq_right[A, B]: fn(Parser[A], Parser[B]) -> Parser[B]
- seq_pair[A, B]: fn(Parser[A], Parser[B]) -> Parser[(A, B)]
- between[A, B, C]: fn(Parser[A], Parser[B], Parser[C]) -> Parser[B]
```

**Kira Features Exercised**:
- Multiple generic type parameters
- Higher-order functions taking functions
- Tuple types

### Task 3.2: Alternation Combinators
**File**: `src/combinators.ki`
**Status**: [x] Complete
**Dependencies**: Task 3.1

Implement choice combinators:

```
Combinators:
- or_else[T]: fn(Parser[T], Parser[T]) -> Parser[T]
- choice[T]: fn(List[Parser[T]]) -> Parser[T]
- optional[T]: fn(Parser[T]) -> Parser[Option[T]]
```

### Task 3.3: Transformation Combinators
**File**: `src/combinators.ki`
**Status**: [x] Complete
**Dependencies**: Task 3.1

Implement functor/applicative operations:

```
Combinators:
- map[A, B]: fn(Parser[A], fn(A) -> B) -> Parser[B]
- map2[A, B, C]: fn(Parser[A], Parser[B], fn(A, B) -> C) -> Parser[C]
- map3[A, B, C, D]: fn(Parser[A], Parser[B], Parser[C], fn(A, B, C) -> D) -> Parser[D]
- replace[A, B]: fn(Parser[A], B) -> Parser[B]
```

### Task 3.4: Repetition Combinators
**File**: `src/combinators.ki`
**Status**: [x] Complete
**Dependencies**: Tasks 3.1-3.3

Implement repetition:

```
Combinators:
- many[T]: fn(Parser[T]) -> Parser[List[T]]
- many1[T]: fn(Parser[T]) -> Parser[List[T]]
- count[T]: fn(i32, Parser[T]) -> Parser[List[T]]
- sep_by[T, S]: fn(Parser[T], Parser[S]) -> Parser[List[T]]
- sep_by1[T, S]: fn(Parser[T], Parser[S]) -> Parser[List[T]]
- end_by[T, S]: fn(Parser[T], Parser[S]) -> Parser[List[T]]
```

**Kira Features Exercised**:
- Mutable variables (`var`)
- Loops
- Recursion

### Task 3.5: Lookahead and Backtracking
**File**: `src/combinators.ki`
**Status**: [x] Complete
**Dependencies**: Task 3.1

Implement lookahead:

```
Combinators:
- lookahead[T]: fn(Parser[T]) -> Parser[T]
- not_followed_by[T]: fn(Parser[T]) -> Parser[void]
- try_p[T]: fn(Parser[T]) -> Parser[T]
```

### Task 3.6: Error Enhancement Combinators
**File**: `src/combinators.ki`
**Status**: [x] Complete
**Dependencies**: Task 3.1

Implement error handling:

```
Combinators:
- label[T]: fn(Parser[T], string) -> Parser[T]
- context[T]: fn(Parser[T], string) -> Parser[T]
```

### Task 3.7: Recursive Parser Support
**File**: `src/combinators.ki`
**Status**: [x] Complete
**Dependencies**: Task 3.1

Implement lazy evaluation for recursion:

```
Types:
- LazyParser[T] (product type with thunk)

Combinators:
- lazy[T]: fn(fn() -> Parser[T]) -> Parser[T]
- fix[T]: fn(fn(Parser[T]) -> Parser[T]) -> Parser[T]
```

### Task 3.8: Combinators Module Tests
**File**: `tests/test_combinators.ki`
**Status**: [x] Complete
**Dependencies**: Tasks 3.1-3.7

```
Tests:
Sequencing:
- test_and_then_success
- test_and_then_first_fails
- test_and_then_second_fails
- test_seq_left_keeps_left
- test_seq_right_keeps_right
- test_seq_pair_keeps_both
- test_between_extracts_middle

Alternation:
- test_or_else_first_succeeds
- test_or_else_second_succeeds
- test_or_else_both_fail_merges_errors
- test_choice_empty_list
- test_choice_first_match
- test_choice_later_match
- test_optional_present
- test_optional_absent

Transformation:
- test_map_transforms_value
- test_map2_combines_two
- test_map3_combines_three
- test_replace_replaces_value

Repetition:
- test_many_zero_matches
- test_many_multiple_matches
- test_many1_requires_one
- test_many1_empty_fails
- test_count_exact
- test_sep_by_empty
- test_sep_by_one_item
- test_sep_by_multiple_items
- test_sep_by1_requires_one
- test_end_by_with_trailing

Lookahead:
- test_lookahead_no_consume
- test_not_followed_by_success
- test_not_followed_by_failure
- test_try_p_backtracks

Error:
- test_label_replaces_expected
- test_context_adds_context

Recursion:
- test_lazy_defers_construction
- test_fix_recursive_parser
```

---

## Phase 4: Lexer Helpers

### Task 4.1: Whitespace Handling
**File**: `src/lexer.ki`
**Status**: [x] Complete
**Dependencies**: Phase 3 complete

Implement whitespace utilities:

```
Functions:
- skip_ws: fn() -> Parser[void]
- lexeme[T]: fn(Parser[T]) -> Parser[T]
```

### Task 4.2: Token Parsers
**File**: `src/lexer.ki`
**Status**: [x] Complete
**Dependencies**: Task 4.1

Implement common token parsers:

```
Parsers:
- keyword: fn(string) -> Parser[string]
- symbol: fn(string) -> Parser[string]
- identifier: fn() -> Parser[string]
- integer: fn() -> Parser[i64]
- string_literal: fn() -> Parser[string]
```

### Task 4.3: Lexer Module Tests
**File**: `tests/test_lexer.ki`
**Status**: [x] Complete
**Dependencies**: Tasks 4.1-4.2

```
Tests:
- test_skip_ws_empty
- test_skip_ws_spaces
- test_skip_ws_mixed_whitespace
- test_lexeme_strips_trailing_ws
- test_keyword_matches_word
- test_keyword_not_prefix
- test_symbol_matches
- test_identifier_simple
- test_identifier_with_numbers
- test_identifier_leading_number_fails
- test_integer_positive
- test_integer_multidigit
- test_string_literal_simple
- test_string_literal_escaped_chars
- test_string_literal_empty
```

---

## Phase 5: Expression Parsing

### Task 5.1: Operator Types
**File**: `src/expr.ki`
**Status**: [x] Complete
**Dependencies**: Phase 3 complete

Implement operator infrastructure:

```
Types:
- Assoc (sum type: AssocLeft, AssocRight, AssocNone)
- Operator[T] (product type with parser and assoc)
- PrecLevel[T] (type alias for List[Operator[T]])
```

### Task 5.2: Chain Combinators
**File**: `src/expr.ki`
**Status**: [x] Complete
**Dependencies**: Task 5.1

Implement chaining for operators:

```
Combinators:
- chainl1[T]: fn(Parser[T], Parser[fn(T, T) -> T]) -> Parser[T]
- chainl1_rest[T]: fn(T, Parser[T], Parser[fn(T, T) -> T]) -> Parser[T]
- chainr1[T]: fn(Parser[T], Parser[fn(T, T) -> T]) -> Parser[T]
```

### Task 5.3: Expression Parser Builder
**File**: `src/expr.ki`
**Status**: [x] Complete
**Dependencies**: Task 5.2

Implement precedence table parsing:

```
Functions:
- add_operators[T]: fn(Parser[T], PrecLevel[T]) -> Parser[T]
- build_expr_parser[T]: fn(List[PrecLevel[T]], Parser[T]) -> Parser[T]
```

### Task 5.4: Expression Module Tests
**File**: `tests/test_expr.ki`
**Status**: [x] Complete
**Dependencies**: Tasks 5.1-5.3

```
Tests:
- test_chainl1_single_term
- test_chainl1_two_terms
- test_chainl1_left_associative
- test_chainr1_right_associative
- test_add_operators_single_level
- test_build_expr_parser_precedence
- test_build_expr_parser_associativity
```

---

## Phase 6: IO Integration

### Task 6.1: File Parsing Types
**File**: `src/io.ki`
**Status**: [x] Complete
**Dependencies**: Phase 3 complete

Implement IO error types:

```
Types:
- ParseFileError (sum type: IoError, ParseError)

Functions:
- format_error: effect fn(ParseFileError) -> IO[string]
```

**Kira Features Exercised**:
- Effect functions
- IO type

### Task 6.2: File Parsing Functions
**File**: `src/io.ki`
**Status**: [x] Complete
**Dependencies**: Task 6.1

Implement file parsing:

```
Functions:
- parse_file[T]: effect fn(string, Parser[T]) -> IO[Result[T, ParseFileError]]
```

---

## Phase 7: Main Module and Re-exports

### Task 7.1: Main PCL Module
**File**: `src/pcl.ki`
**Status**: [x] Complete
**Dependencies**: Phases 1-6 complete

Create main module with public API:

```
Re-exports:
- All types from types.ki
- All primitives from primitives.ki
- All combinators from combinators.ki
- All lexer helpers from lexer.ki
- All expression utilities from expr.ki
- All IO functions from io.ki
```

---

## Phase 8: Example Implementations

### Task 8.1: JSON Parser
**File**: `examples/json.ki`
**Status**: [x] Complete
**Dependencies**: Phases 1-4 complete

Implement complete JSON parser:

```
Types:
- JsonValue (sum type: JsonNull, JsonBool, JsonNumber, JsonString, JsonArray, JsonObject)

Parsers:
- json_null: fn() -> Parser[JsonValue]
- json_bool: fn() -> Parser[JsonValue]
- json_number: fn() -> Parser[JsonValue]
- json_string: fn() -> Parser[JsonValue]
- json_array: fn() -> Parser[JsonValue]
- json_object: fn() -> Parser[JsonValue]
- json_value: fn() -> Parser[JsonValue]

Public API:
- parse_json: fn(string) -> Result[JsonValue, ParseError]
```

### Task 8.2: JSON Parser Tests
**File**: `tests/test_json.ki`
**Status**: [x] Complete
**Dependencies**: Task 8.1

```
Tests:
- test_parse_null
- test_parse_true
- test_parse_false
- test_parse_integer
- test_parse_negative_number
- test_parse_string_simple
- test_parse_string_escaped
- test_parse_empty_array
- test_parse_array_one_element
- test_parse_array_multiple
- test_parse_nested_array
- test_parse_empty_object
- test_parse_object_one_pair
- test_parse_object_multiple
- test_parse_nested_object
- test_parse_complex_json
- test_parse_whitespace_handling
- test_parse_error_unclosed_array
- test_parse_error_unclosed_object
```

### Task 8.3: Calculator Example
**File**: `examples/calc.ki`
**Status**: [x] Complete
**Dependencies**: Phases 1-5 complete

Implement expression parser and evaluator:

```
Types:
- Expr (sum type: Literal, Add, Sub, Mul, Div, Neg, Parens)

Parsers:
- literal: fn() -> Parser[Expr]
- parens: fn() -> Parser[Expr]
- negation: fn() -> Parser[Expr]
- factor: fn() -> Parser[Expr]
- term: fn() -> Parser[Expr]
- expr: fn() -> Parser[Expr]

Functions:
- parse_expr: fn(string) -> Result[Expr, ParseError]
- eval: fn(Expr) -> i64
```

### Task 8.4: Calculator Tests
**File**: (within `tests/test_expr.ki` or separate)
**Status**: [x] Complete
**Dependencies**: Task 8.3

```
Tests:
- test_eval_literal
- test_eval_addition
- test_eval_subtraction
- test_eval_multiplication
- test_eval_division
- test_eval_negation
- test_eval_parentheses
- test_eval_precedence_mul_add
- test_eval_precedence_parens
- test_eval_left_associative
- test_parse_and_eval_complex
```

### Task 8.5: S-Expression Parser (Optional)
**File**: `examples/lisp.ki`
**Status**: [x] Complete
**Dependencies**: Phases 1-4 complete

Implement S-expression parser:

```
Types:
- SExpr (sum type: Symbol, Number, SList)

Parsers:
- symbol: fn() -> Parser[SExpr]
- number: fn() -> Parser[SExpr]
- slist: fn() -> Parser[SExpr]
- sexpr: fn() -> Parser[SExpr]

Public API:
- parse_sexpr: fn(string) -> Result[SExpr, ParseError]
```

---

## Implementation Order Summary

```
Phase 1: Foundation (Core Types)
  1.1 Project Structure
  1.2 Core Types Module
  1.3 String Utilities
  1.4 List Utilities
  1.5 Types Tests

Phase 2: Primitive Parsers
  2.1 Basic Character Parsers
  2.2 Character Class Parsers
  2.3 String and Terminal Parsers
  2.4 Parser Execution
  2.5 Primitives Tests

Phase 3: Core Combinators
  3.1 Sequencing
  3.2 Alternation
  3.3 Transformation
  3.4 Repetition
  3.5 Lookahead
  3.6 Error Enhancement
  3.7 Recursive Support
  3.8 Combinators Tests

Phase 4: Lexer Helpers
  4.1 Whitespace
  4.2 Token Parsers
  4.3 Lexer Tests

Phase 5: Expression Parsing
  5.1 Operator Types
  5.2 Chain Combinators
  5.3 Expression Builder
  5.4 Expression Tests

Phase 6: IO Integration
  6.1 File Parsing Types
  6.2 File Parsing Functions

Phase 7: Main Module
  7.1 PCL Module with Re-exports

Phase 8: Examples
  8.1 JSON Parser
  8.2 JSON Tests
  8.3 Calculator
  8.4 Calculator Tests
  8.5 S-Expression Parser (optional)
```

---

## Milestones

### M1: Core Foundation Complete
**Criteria**: Phases 1-2 complete with tests passing
**Deliverable**: Can parse individual characters and strings

### M2: Combinator Library Complete
**Criteria**: Phase 3 complete with tests passing
**Deliverable**: Full combinator API available

### M3: Production-Ready Library
**Criteria**: Phases 4-7 complete with tests passing
**Deliverable**: Complete PCL library with lexer helpers and expression parsing

### M4: Examples and Documentation
**Criteria**: Phase 8 complete
**Deliverable**: Working JSON parser, calculator, documentation

---

## Testing Strategy

### Unit Tests
- Each module has corresponding test file
- Test happy path and error cases
- Test edge cases (empty input, single character, etc.)

### Property Tests
- `many` preserves: length(output) >= 0
- `many1` preserves: length(output) >= 1
- `or_else` is commutative for disjoint parsers
- `and_then` is associative
- `map(p, id) == p`

### Integration Tests
- JSON parser on various JSON files
- Calculator on complex expressions
- Error message quality tests

---

## Notes

### Potential Challenges

1. **Recursion**: The `lazy` combinator is critical for recursive grammars. Test early.

2. **Performance**: `many` uses a loop with mutable state. Ensure this is idiomatic Kira.

3. **Error Merging**: The `merge_errors` function must correctly identify the furthest error position.

4. **Generic Type Inference**: Kira requires explicit type parameters. May need to specify at call sites.

### Design Decisions to Revisit

1. Should `ParseError` include source snippet for better error messages?

2. Should we add `map4`, `map5` or use a different approach for many-argument mapping?

3. Should `chainl1`/`chainr1` use an accumulator or pure recursion?

4. Do we need `manyTill` or other specialized repetition combinators?

---

## Dependencies

```
src/types.ki      <- (foundation)
src/primitives.ki <- src/types.ki
src/combinators.ki <- src/types.ki, src/primitives.ki
src/lexer.ki      <- src/combinators.ki
src/expr.ki       <- src/combinators.ki
src/io.ki         <- src/types.ki, std.fs
src/pcl.ki        <- (all modules)
examples/*.ki     <- src/pcl.ki
tests/*.ki        <- corresponding src modules
```
