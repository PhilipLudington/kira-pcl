# /kira-safety

Perform a security-focused review of Kira code.

## Usage

```
/kira-safety [path]
```

- No argument: Review all `.ki` files in project
- With path: Review specific file or directory

## What This Command Does

Analyzes code for security issues:
1. Input validation
2. Path traversal vulnerabilities
3. Command injection risks
4. Secret exposure
5. Integer overflow risks
6. Effect boundary violations

## Instructions for Claude

When the user runs `/kira-safety`:

1. **Find files to review**:
   - If path specified, review that file/directory
   - Otherwise, find all `.ki` files in project
   - Focus on effect functions (IO boundaries)

2. **Check security categories**:

### Input Validation (Critical)
- [ ] All external input validated before use
- [ ] Length limits enforced
- [ ] Format validation for structured data
- [ ] Whitelist approach preferred over blacklist

```kira
// GOOD: Input validated
let process: fn(string) -> Result[Data, Error] = fn(input: string) -> Result[Data, Error] {
    if std.string.length(input) > MAX_LENGTH {
        return Err(Error.TooLong)
    }
    if not is_valid_format(input) {
        return Err(Error.InvalidFormat)
    }
    return parse(input)
}

// BAD: No validation
let process: fn(string) -> Data = fn(input: string) -> Data {
    return parse(input)  // Direct use without validation
}
```

### Path Traversal (Critical)
- [ ] File paths validated for traversal attacks
- [ ] Canonical paths compared against base directory
- [ ] User-provided paths sanitized

```kira
// Look for patterns like:
effect fn read_user_file(user_path: string) -> IO[Result[string, Error]] {
    // Should check for ".." and validate canonical path
}
```

### Command Injection (Critical)
- [ ] No untrusted input passed to shell commands
- [ ] Argument arrays used instead of string concatenation

```kira
// BAD: Command injection risk
effect fn run_command(user_input: string) -> IO[void] {
    system("process " + user_input)  // DANGEROUS
}

// GOOD: Use argument arrays
effect fn run_command(filename: string) -> IO[Result[void, Error]] {
    if not is_safe_filename(filename) {
        return Err(Error.InvalidFilename)
    }
    exec("process", [filename])
}
```

### Secret Exposure (Critical)
- [ ] No secrets in logs
- [ ] No hardcoded credentials
- [ ] Sensitive data not returned in error messages

```kira
// BAD: Secret in log
effect fn authenticate(password: string) -> IO[void] {
    std.log.debug("Password: {password}")  // NEVER!
}

// BAD: Hardcoded secret
let API_KEY: string = "sk-1234567890"  // NEVER!
```

### Integer Safety (Warning)
- [ ] Checked/saturating arithmetic for external input
- [ ] No unchecked casts from larger to smaller types

### Effect Boundaries (Warning)
- [ ] Pure functions don't bypass purity through FFI or unsafe
- [ ] Effect boundaries clearly defined

3. **Generate security report**:

```
# Kira-Toolkit Security Report

## Summary
- Files reviewed: N
- Security issues: N (X critical, Y warnings)
- Security score: XX/100

## Critical Issues

### [S1] Command Injection Risk
**File:** src/cli.ki:45
**Severity:** Critical
**Issue:** User input passed directly to shell command
**Fix:** Use argument arrays and validate input

### [S2] Missing Input Validation
**File:** src/api.ki:23
**Severity:** Critical
**Issue:** External input used without validation
**Fix:** Add length and format validation

## Warnings

### [W1] Integer Overflow Risk
**File:** src/math.ki:67
**Severity:** Warning
**Issue:** Unchecked arithmetic on user-provided values
**Fix:** Use checked_add or saturating_add

## Recommendations

1. Add input validation layer at all external boundaries
2. Use typed wrappers for validated data
3. Audit all effect functions for security implications
```

4. **Provide security recommendations**:

For each issue, provide specific remediation steps.

## Security Checklist

Before deployment:
- [ ] All external input validated
- [ ] No path traversal vulnerabilities
- [ ] No command injection risks
- [ ] No secrets in code or logs
- [ ] Integer overflow handled
- [ ] Effect boundaries secure
