# /kira-review

Review Kira code against Kira-Toolkit standards.

## Usage

```
/kira-review [path]
```

- No argument: Review all `.ki` files in project
- With path: Review specific file or directory

## What This Command Does

Analyzes code for compliance with STANDARDS.md, checking:
1. Naming conventions
2. Purity and effects
3. Pattern matching
4. Algebraic data types
5. Error handling
6. API design
7. Documentation
8. Code organization

## Instructions for Claude

When the user runs `/kira-review`:

1. **Find files to review**:
   - If path specified, review that file/directory
   - Otherwise, find all `.ki` files in `src/` and project root
   - Exclude `tests/`, `build/`, `target/`

2. **Check each category**:

### Naming Conventions
- [ ] Types are `PascalCase`
- [ ] Functions are `snake_case`
- [ ] Variables are `snake_case`
- [ ] Constants are `UPPER_SNAKE_CASE`
- [ ] Sum type variants are `PascalCase`
- [ ] Boolean names start with `is_`, `has_`, `can_`, etc.
- [ ] Acronyms treated as words (Http, not HTTP)

### Purity and Effects
- [ ] Pure functions don't call effectful functions
- [ ] Effect functions marked with `effect fn`
- [ ] Effect types correctly used (`IO[T]`, `IO[Result[T, E]]`)
- [ ] Main function is properly an effect function

### Pattern Matching
- [ ] All pattern matches are exhaustive
- [ ] Specific patterns before general ones
- [ ] Wildcard `_` used for unused bindings
- [ ] Guards used appropriately

### Algebraic Data Types
- [ ] Sum types for "one of" relationships
- [ ] Product types for "all of" relationships
- [ ] Types make illegal states unrepresentable
- [ ] Generic types have meaningful parameters

### Error Handling
- [ ] Fallible functions return `Result[T, E]`
- [ ] `?` used for error propagation
- [ ] Custom error types include context
- [ ] Errors not silently swallowed
- [ ] Option used for absent values, not errors

### API Design
- [ ] Functions have ≤4 parameters (or use config records)
- [ ] Return types are appropriate
- [ ] Visibility is minimal (private by default)

### Documentation
- [ ] Public functions have doc comments
- [ ] Parameters documented when non-obvious
- [ ] Return values and errors documented
- [ ] Module has doc comment

### Code Organization
- [ ] Files under 500 lines
- [ ] Functions under 50 lines
- [ ] Imports organized (std, third-party, project)

3. **Generate report**:

```
# Kira-Toolkit Review Report

## Summary
- Files reviewed: N
- Issues found: N (X critical, Y warnings, Z suggestions)
- Compliance: XX%

## Critical Issues
[Issues that likely cause bugs or break purity]

## Warnings
[Standard violations that should be fixed]

## Suggestions
[Best practice improvements]

## Per-File Details

### src/main.ki
- Line 42: Function `getData` should be `get_data` (naming)
- Line 67: Pure function calling effectful code (purity violation)
- Line 89: Non-exhaustive pattern match (pattern matching)

### src/lib.ki
- Line 15: Function has 6 parameters, use config record (api-design)
- Line 30: Missing `effect fn` on IO function (purity)
```

4. **Provide fix suggestions**:

For each issue, suggest the fix:
```
Line 42: `let getData: fn(...)` → `let get_data: fn(...)`
Line 67: Move IO to effect function, or remove print from pure function
Line 89: Add missing pattern: `None => { ... }`
```

## Severity Levels

- **Critical**: Purity violations, non-exhaustive matches, likely bugs
- **Warning**: Standard violations that should be addressed
- **Suggestion**: Best practice improvements, style issues
