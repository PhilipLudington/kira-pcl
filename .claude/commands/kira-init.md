# /kira-init

Create a new Kira-Toolkit project with proper structure and configuration.

## Usage

```
/kira-init [project-name]
```

## What This Command Does

1. Creates project directory structure
2. Sets up initial files
3. Configures Claude Code integration
4. Creates sample Kira code

## Instructions for Claude

When the user runs `/kira-init`:

### 1. Get project name

If not provided, ask for the project name:

```
What would you like to name your Kira project?
```

### 2. Create directory structure

```bash
mkdir -p <project-name>/src
mkdir -p <project-name>/tests
mkdir -p <project-name>/.claude/commands
mkdir -p <project-name>/.claude/rules
```

### 3. Create main.ki

Create `src/main.ki`:

```kira
//! Main entry point for the application.

module main

import std.io.{ println }

/// Application entry point.
effect fn main() -> IO[void] {
    println("Hello, Kira!")
    return
}
```

### 4. Create lib.ki (optional library module)

Create `src/lib.ki`:

```kira
//! Library module for the application.

module lib

import std.option.{ Option, Some, None }
import std.result.{ Result, Ok, Err }

/// Example pure function.
pub let greet: fn(string) -> string = fn(name: string) -> string {
    return "Hello, {name}!"
}

/// Example function returning Option.
pub let find_positive: fn(i32) -> Option[i32] = fn(n: i32) -> Option[i32] {
    if n > 0 {
        return Some(n)
    }
    return None
}
```

### 5. Create test file

Create `tests/test_lib.ki`:

```kira
//! Tests for the lib module.

module test_lib

import lib.{ greet, find_positive }
import std.option.{ Some, None }

let test_greet_returns_greeting: fn() -> void = fn() -> void {
    let result: string = greet("World")
    assert(result == "Hello, World!")
    return
}

let test_find_positive_with_positive: fn() -> void = fn() -> void {
    let result: Option[i32] = find_positive(5)
    assert(is_some(result))
    return
}

let test_find_positive_with_negative: fn() -> void = fn() -> void {
    let result: Option[i32] = find_positive(-3)
    assert(is_none(result))
    return
}
```

### 6. Create README.md

Create `README.md`:

```markdown
# <project-name>

A Kira project using Kira-Toolkit standards.

## Project Structure

```
<project-name>/
├── src/
│   ├── main.ki    # Entry point
│   └── lib.ki     # Library module
├── tests/
│   └── test_lib.ki
└── README.md
```

## Development

This project follows Kira-Toolkit standards. Key commands:

| Command | Purpose |
|---------|---------|
| `/kira-review` | Review code against standards |
| `/kira-safety` | Security-focused review |
| `/kira-check` | Run build and tests |

## Kira Features Used

- Pure functions by default
- Explicit effect tracking
- Pattern matching
- Algebraic data types
```

### 7. Create CLAUDE.md

Create `CLAUDE.md`:

```markdown
## Kira Development

This project uses the Kira-Toolkit framework (v0.1.0) for Kira development standards.

See `kira-toolkit/KIRATOOLKIT.md` for coding guidelines and available commands.

### Key Syntax Requirements
- **Explicit types**: `let x: i32 = 42`
- **Explicit return**: `return value`
- **Pure by default**: Use `effect fn` for IO
- **Pattern matching**: Must be exhaustive
```

### 8. Report completion

```markdown
# Kira Project Created

**Project:** <project-name>

## Structure Created

```
<project-name>/
├── src/
│   ├── main.ki       # Entry point with hello world
│   └── lib.ki        # Sample library module
├── tests/
│   └── test_lib.ki   # Sample tests
├── README.md
└── CLAUDE.md
```

## Next Steps

1. Install Kira-Toolkit: `/kira-install`
2. Review code: `/kira-review`
3. Run tests: `/kira-check`

## Key Kira Concepts

- **Pure functions**: Default, no side effects
- **Effect functions**: Use `effect fn` for IO
- **Pattern matching**: Always exhaustive
- **Explicit types**: No type inference
```
