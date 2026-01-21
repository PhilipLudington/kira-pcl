# /kira-install

Install the Kira-Toolkit Kira development framework into the current project.

## Usage

```
/kira-install
```

## What This Command Does

1. Clones Kira-Toolkit repository
2. Copies rules and commands to `.claude/`
3. Sets up version tracking
4. Updates CLAUDE.md with framework reference

## Instructions for Claude

When the user runs `/kira-install`:

### 1. Clone Kira-Toolkit

Clone the repository into the project:

```bash
git clone https://github.com/PhilipLudington/Kira-Toolkit.git kira-toolkit
rm -rf kira-toolkit/.git
```

### 2. Copy Claude Code integration

Create directories and copy files:

```bash
# Create directories
mkdir -p .claude/commands .claude/rules

# Copy commands
cp kira-toolkit/commands/*.md .claude/commands/

# Copy rules
cp kira-toolkit/rules/*.md .claude/rules/
```

### 3. Set version tracking

Create version file:

```bash
echo "0.1.0" > .claude/kira-toolkit-version
```

### 4. Add Kira-Toolkit reference to CLAUDE.md

If `./CLAUDE.md` doesn't exist, create it. Add the following:

```markdown
## Kira Development

This project uses the Kira-Toolkit framework (v0.1.0) for Kira development standards.

See `kira-toolkit/KIRATOOLKIT.md` for coding guidelines and available commands.

### Key Syntax Requirements
- **Explicit types**: `let x: i32 = 42`
- **Explicit return**: `return value` (not implicit last expression)
- **Pure by default**: Use `effect fn` for IO operations
- **Pattern matching**: All matches must be exhaustive
- **Closures with full types**: `fn(x: i32) -> i32 { return x * 2 }`
```

### 5. Verify installation

Check that all files were copied:

```bash
ls .claude/commands/
ls .claude/rules/
cat .claude/kira-toolkit-version
```

Expected files:
- `.claude/commands/`: kira-init.md, kira-install.md, kira-review.md, kira-safety.md, kira-check.md, kira-update.md
- `.claude/rules/`: kira-naming.md, kira-purity.md, kira-pattern-matching.md, kira-adt.md, kira-errors.md, kira-api-design.md, kira-testing.md, kira-logging.md, kira-security.md

### 6. Report completion

```markdown
# Kira-Toolkit Installation Complete

**Version:** 0.1.0

## Installed Components

### Commands (6)
- `/kira-init` - Create new Kira-Toolkit project
- `/kira-install` - Install Kira-Toolkit (this command)
- `/kira-review` - Review code against standards
- `/kira-safety` - Security-focused review
- `/kira-check` - Run validation tooling
- `/kira-update` - Update to latest version

### Rules (9)
- kira-naming - Naming conventions
- kira-purity - Pure vs effect functions
- kira-pattern-matching - Exhaustive matching
- kira-adt - Algebraic data types
- kira-errors - Error handling
- kira-api-design - API design patterns
- kira-testing - Testing standards
- kira-logging - Logging practices
- kira-security - Security rules

### Documentation (in kira-toolkit/docs/)
- Pattern guides: purity, pattern-matching, adt, higher-order, errors, api-design
- Security guides: validation, injection

## Next Steps

1. Review `kira-toolkit/STANDARDS.md` for full coding standards
2. Use `/kira-review` to check existing code
3. Use `/kira-init` to create new compliant projects

## Key Kira Features

- Pure functions by default
- Explicit effect tracking with `effect fn`
- Algebraic data types (sum and product types)
- Exhaustive pattern matching
- No type inference - all types explicit
```

## After Installation

The following commands are now available:

| Command | Purpose |
|---------|---------|
| `/kira-review` | Review code against standards |
| `/kira-safety` | Security-focused review |
| `/kira-check` | Run validation tooling |
| `/kira-update` | Update to latest version |

## Updating

To update Kira-Toolkit to the latest version:

```
/kira-update
```
