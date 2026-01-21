# /kira-check

Run build, tests, and validation checks on Kira code.

## Usage

```
/kira-check [--build] [--test] [--lint]
```

- No flags: Run all checks
- `--build`: Only run build
- `--test`: Only run tests
- `--lint`: Only run linting

## What This Command Does

1. Validates project structure
2. Runs the Kira compiler (if available)
3. Executes test suite
4. Performs static analysis

## Instructions for Claude

When the user runs `/kira-check`:

### 1. Validate project structure

Check for expected directories and files:

```bash
# Check structure
ls -la src/
ls -la tests/
```

Expected structure:
```
project/
├── src/
│   ├── main.ki or lib.ki
│   └── *.ki
├── tests/
│   └── test_*.ki
└── README.md
```

### 2. Find all Kira files

```bash
find . -name "*.ki" -type f | grep -v node_modules | grep -v target
```

### 3. Validate file contents

For each `.ki` file, check:

**Syntax Requirements:**
- [ ] All `let` bindings have explicit types
- [ ] All functions have explicit return types
- [ ] All functions use explicit `return` statements
- [ ] Effect functions marked with `effect fn`

**Pattern Matching:**
- [ ] All `match` statements appear exhaustive
- [ ] No obvious missing cases

**Type Consistency:**
- [ ] Generic types use consistent naming
- [ ] Type parameters properly specified

### 4. Run tests (if test runner available)

```bash
# Example: if Kira has a test runner
kira test tests/

# Or check test file structure
ls tests/test_*.ki
```

### 5. Check for common issues

**Critical Issues:**
- Pure functions calling effect functions
- Non-exhaustive pattern matches
- Missing type annotations

**Warnings:**
- Files over 500 lines
- Functions over 50 lines
- Missing documentation on public items

**Style Issues:**
- Naming convention violations
- Import organization

### 6. Generate report

```
# Kira-Toolkit Check Report

## Project Status

| Check | Status |
|-------|--------|
| Project Structure | ✓ Pass |
| Syntax Validation | ✓ Pass |
| Type Annotations | ✓ Pass |
| Pattern Matching | ⚠ 1 Warning |
| Tests | ✓ 5/5 Passing |

## Summary

- Files checked: 8
- Issues found: 1
- Tests: 5 passing, 0 failing

## Issues

### Warning: Possible non-exhaustive match
**File:** src/handler.ki:45
**Issue:** Match on Result type may be missing error case
**Suggestion:** Verify all error variants are handled

## Test Results

```
test_lib.ki
  ✓ test_greet_returns_greeting
  ✓ test_find_positive_with_positive
  ✓ test_find_positive_with_negative

test_main.ki
  ✓ test_main_runs
  ✓ test_config_loads
```

## Next Steps

1. Review warning in src/handler.ki:45
2. Add tests for uncovered functions
3. Run `/kira-safety` for security review
```

### 7. Exit status

Report overall status:
- **Pass**: All checks passed
- **Warning**: Minor issues found
- **Fail**: Critical issues found

## Quick Checks

For fast validation during development:

```
/kira-check --lint   # Just check style/syntax
/kira-check --test   # Just run tests
/kira-check --build  # Just validate compilation
```
