# Injection Prevention Patterns

Patterns for preventing injection attacks in Kira.

## The Injection Threat Model

Injection attacks occur when:
1. Untrusted input is incorporated into a command/query
2. The input can change the intended meaning/structure
3. The system executes the modified command

Common injection types:
- Command injection (shell commands)
- Path injection (directory traversal)
- SQL injection (database queries)
- Format string injection

## Pattern 1: Command Injection Prevention

Never concatenate user input into shell commands:

```kira
// BAD: Command injection vulnerability
effect fn bad_process_file(filename: string) -> IO[void] {
    // User could input: "file.txt; rm -rf /"
    system("cat " + filename)
    return
}

// GOOD: Use argument arrays
effect fn safe_process_file(filename: string) -> IO[Result[void, Error]] {
    // First validate the filename
    if not is_safe_filename(filename) {
        return Err(Error.InvalidFilename)
    }
    // Use exec with argument array - shell metacharacters are not interpreted
    return exec("cat", [filename])
}

// Safe filename validation
let is_safe_filename: fn(string) -> bool = fn(name: string) -> bool {
    // Whitelist approach: only allow alphanumeric, dot, dash, underscore
    for c in std.string.chars(name) {
        if not (is_alphanumeric(c) or c == '.' or c == '-' or c == '_') {
            return false
        }
    }
    // Don't allow empty or starting with dot (hidden files)
    if std.string.length(name) == 0 {
        return false
    }
    if std.string.starts_with(name, ".") {
        return false
    }
    return true
}
```

## Pattern 2: Path Traversal Prevention

Prevent attackers from accessing files outside allowed directories:

```kira
type PathError =
    | TraversalAttempt
    | OutsideBaseDir
    | InvalidPath(string)

let safe_path: fn(string, string) -> Result[string, PathError] =
    fn(base_dir: string, user_path: string) -> Result[string, PathError] {
        // Check for obvious traversal attempts
        if std.string.contains(user_path, "..") {
            return Err(PathError.TraversalAttempt)
        }
        if std.string.contains(user_path, "~") {
            return Err(PathError.TraversalAttempt)
        }
        if std.string.starts_with(user_path, "/") {
            return Err(PathError.TraversalAttempt)
        }

        // Join paths
        let full_path: string = join_path(base_dir, user_path)

        // Canonicalize to resolve any remaining tricks
        let canonical: string = canonicalize(full_path)?

        // Verify the resolved path is still under base_dir
        let canonical_base: string = canonicalize(base_dir)?
        if not std.string.starts_with(canonical, canonical_base) {
            return Err(PathError.OutsideBaseDir)
        }

        return Ok(canonical)
    }

// Usage
effect fn read_user_document(doc_name: string) -> IO[Result[string, Error]] {
    let base: string = "/app/user_documents"
    match safe_path(base, doc_name) {
        Ok(path) => { return read_file(path) }
        Err(e) => { return Err(Error.PathError(e)) }
    }
}
```

## Pattern 3: SQL Injection Prevention

Never concatenate user input into SQL queries:

```kira
// BAD: SQL injection vulnerability
let bad_find_user: fn(string) -> string = fn(username: string) -> string {
    // User could input: "'; DROP TABLE users; --"
    return "SELECT * FROM users WHERE name = '" + username + "'"
}

// GOOD: Use parameterized queries
type Query = {
    sql: string,
    params: List[QueryParam]
}

type QueryParam =
    | StringParam(string)
    | IntParam(i32)
    | NullParam

let find_user_query: fn(string) -> Query = fn(username: string) -> Query {
    return Query {
        sql: "SELECT * FROM users WHERE name = ?",
        params: Cons(StringParam(username), Nil)
    }
}

// Query executor handles escaping
effect fn execute_query(q: Query) -> IO[Result[List[Row], DbError]] = ...
```

## Pattern 4: HTML/XSS Prevention

Escape user content before including in HTML:

```kira
let escape_html: fn(string) -> string = fn(input: string) -> string {
    let s: string = input
    let s: string = std.string.replace(s, "&", "&amp;")
    let s: string = std.string.replace(s, "<", "&lt;")
    let s: string = std.string.replace(s, ">", "&gt;")
    let s: string = std.string.replace(s, "\"", "&quot;")
    let s: string = std.string.replace(s, "'", "&#x27;")
    let s: string = std.string.replace(s, "/", "&#x2F;")
    return s
}

// Typed wrapper for safe HTML
type SafeHtml = { content: string }

let safe_text: fn(string) -> SafeHtml = fn(text: string) -> SafeHtml {
    return SafeHtml { content: escape_html(text) }
}

let safe_literal: fn(string) -> SafeHtml = fn(html: string) -> SafeHtml {
    // Only for trusted HTML literals
    return SafeHtml { content: html }
}

// Templates only accept SafeHtml
let render_user_comment: fn(SafeHtml) -> string = fn(comment: SafeHtml) -> string {
    return "<div class=\"comment\">" + comment.content + "</div>"
}

// Usage
let user_input: string = "<script>alert('xss')</script>"
let safe_comment: SafeHtml = safe_text(user_input)  // Escaped
let html: string = render_user_comment(safe_comment)
// Result: <div class="comment">&lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt;</div>
```

## Pattern 5: URL Parameter Safety

Encode user input in URLs:

```kira
let url_encode: fn(string) -> string = fn(input: string) -> string {
    // Encode special characters
    var result: string = ""
    for c in std.string.chars(input) {
        if is_alphanumeric(c) or c == '-' or c == '_' or c == '.' or c == '~' {
            result = result + std.string.from_char(c)
        } else {
            result = result + "%" + to_hex(c)
        }
    }
    return result
}

// Build URLs safely
let build_search_url: fn(string, string) -> string =
    fn(base_url: string, query: string) -> string {
        return base_url + "?q=" + url_encode(query)
    }
```

## Pattern 6: Format String Safety

Don't let users control format strings:

```kira
// BAD: User-controlled format string
let bad_log: fn(string, string) -> string = fn(format: string, data: string) -> string {
    return std.string.format(format, data)  // User could exploit format specifiers
}

// GOOD: Fixed format, user provides data only
let safe_log: fn(string) -> string = fn(message: string) -> string {
    return "[LOG] " + sanitize_for_log(message)
}

let sanitize_for_log: fn(string) -> string = fn(input: string) -> string {
    // Remove control characters, newlines, etc.
    let filtered: List[char] = filter[char](
        std.string.chars(input),
        fn(c: char) -> bool {
            return is_printable(c) and c != '\n' and c != '\r'
        }
    )
    return std.string.from_chars(filtered)
}
```

## Pattern 7: Type-Safe Command Building

Use types to enforce safe command construction:

```kira
type Command =
    | ListFiles { directory: ValidPath }
    | CopyFile { source: ValidPath, dest: ValidPath }
    | ReadFile { path: ValidPath }

// ValidPath can only be created through validation
type ValidPath = { path: string }

let validate_path: fn(string) -> Result[ValidPath, PathError] = ...

// Command execution uses validated paths
effect fn execute_command(cmd: Command) -> IO[Result[string, Error]] {
    match cmd {
        ListFiles { directory } => {
            return exec("ls", [directory.path])
        }
        CopyFile { source, dest } => {
            return exec("cp", [source.path, dest.path])
        }
        ReadFile { path } => {
            return exec("cat", [path.path])
        }
    }
}

// Usage - must validate first
let user_dir: string = get_user_input()
match validate_path(user_dir) {
    Ok(valid) => {
        let cmd: Command = ListFiles { directory: valid }
        execute_command(cmd)
    }
    Err(e) => {
        return Err(Error.InvalidPath(e))
    }
}
```

## Pattern 8: Defense in Depth

Layer multiple protections:

```kira
effect fn secure_file_operation(user_path: string) -> IO[Result[string, Error]] {
    // Layer 1: Input validation
    if not is_valid_path_format(user_path) {
        return Err(Error.InvalidFormat)
    }

    // Layer 2: Sanitization
    let sanitized: string = sanitize_path(user_path)

    // Layer 3: Path resolution and bounds checking
    let safe: ValidPath = validate_path("/app/data", sanitized)?

    // Layer 4: Permission check
    if not has_read_permission(safe) {
        return Err(Error.PermissionDenied)
    }

    // Layer 5: Audit logging
    log_file_access(safe)

    // Finally perform operation
    return read_file(safe.path)
}
```

## Injection Prevention Checklist

- [ ] Never concatenate user input into commands
- [ ] Use argument arrays for command execution
- [ ] Validate and canonicalize file paths
- [ ] Use parameterized queries for SQL
- [ ] Escape user content in HTML output
- [ ] URL-encode user input in URLs
- [ ] Don't let users control format strings
- [ ] Use typed wrappers for validated data
- [ ] Apply defense in depth (multiple layers)
- [ ] Log suspicious inputs for monitoring
