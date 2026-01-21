# /kira-update

Update Kira-Toolkit to the latest version.

## Usage

```
/kira-update
```

## What This Command Does

1. Checks current installed version
2. Fetches latest version from repository
3. Updates rules and commands
4. Reports changes

## Instructions for Claude

When the user runs `/kira-update`:

### 1. Check current version

```bash
cat .claude/kira-toolkit-version
```

If file doesn't exist, Kira-Toolkit is not installed. Run `/kira-install` first.

### 2. Backup current installation

```bash
# Backup current rules and commands
cp -r .claude/rules .claude/rules.backup
cp -r .claude/commands .claude/commands.backup
```

### 3. Fetch latest Kira-Toolkit

```bash
# Remove old toolkit
rm -rf kira-toolkit

# Clone fresh copy
git clone https://github.com/PhilipLudington/Kira-Toolkit.git kira-toolkit
rm -rf kira-toolkit/.git
```

### 4. Get new version

```bash
# Check new version (from README or version file)
grep -m1 "Version" kira-toolkit/README.md
```

### 5. Update rules and commands

```bash
# Update commands
cp kira-toolkit/commands/*.md .claude/commands/

# Update rules
cp kira-toolkit/rules/*.md .claude/rules/
```

### 6. Update version file

```bash
echo "0.1.0" > .claude/kira-toolkit-version
```

### 7. Compare changes

Check what changed:
- New rules added
- Rules modified
- New commands added
- Commands modified

### 8. Report update status

```markdown
# Kira-Toolkit Update Complete

## Version
- Previous: 0.0.1
- Current: 0.1.0

## Changes

### New Rules
- kira-purity.md - Pure vs effect functions
- kira-pattern-matching.md - Exhaustive matching
- kira-adt.md - Algebraic data types

### Updated Rules
- kira-errors.md - Added pattern matching examples
- kira-testing.md - Added property-based testing section

### New Commands
- (none)

### Updated Commands
- kira-review.md - Added purity checks

## Migration Notes

### Breaking Changes
- (none)

### Recommended Actions
1. Review updated rules in `.claude/rules/`
2. Run `/kira-review` to check code against new standards
3. Update existing code to match new patterns

## Backup Location

Previous version backed up to:
- `.claude/rules.backup/`
- `.claude/commands.backup/`

To restore: `cp -r .claude/rules.backup/* .claude/rules/`
```

### 9. Clean up backups (optional)

After user confirms update is working:

```bash
rm -rf .claude/rules.backup
rm -rf .claude/commands.backup
```

## Rollback

If update causes issues:

```bash
# Restore from backup
cp -r .claude/rules.backup/* .claude/rules/
cp -r .claude/commands.backup/* .claude/commands/
```

## Version History

| Version | Changes |
|---------|---------|
| 0.1.0 | Initial release with purity, pattern matching, ADT rules |
