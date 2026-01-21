---
globs: ["**/*.ki"]
---

# Logging Rules

## Log Levels

| Level | Use For |
|-------|---------|
| `error` | Failures preventing operation |
| `warn` | Unexpected but recoverable |
| `info` | Significant events (startup, config) |
| `debug` | Troubleshooting details |
| `trace` | Detailed execution flow |

## Logging is an Effect

**L1**: Logging performs IO, so it must be in effect functions:

```kira
// Logging is effectful
effect fn process_request(req: Request) -> IO[Response] {
    std.log.info("Processing request: id={req.id}")
    let response: Response = handle(req)
    std.log.info("Request complete: id={req.id}, status={response.status}")
    return response
}

// Pure functions cannot log
// let pure_fn: fn(i32) -> i32 = fn(x: i32) -> i32 {
//     std.log.debug("...")  // ERROR: cannot log in pure function
//     return x
// }
```

## Message Format

**L2**: Include context in messages:

```kira
// GOOD
std.log.error("Failed to load config: path={path}, error={err}")
std.log.info("Server started: port={port}")
std.log.debug("Processing request: id={id}, client={ip}")

// BAD
std.log.error("Load failed")
std.log.info("Started")
```

## What NOT to Log

**L3**: Never log secrets:
- Passwords, tokens, API keys
- Private keys, certificates
- Session identifiers

**L4**: Never log PII without explicit requirement

**L5**: Never log in tight loops:

```kira
// BAD: Millions of log entries
for item in items {
    std.log.debug("Processing {item}")
}

// GOOD: Log summary
std.log.debug("Processing {length(items)} items")
```

**L6**: Never log large data blobs

## Structured Logging

Prefer structured format for machine parsing:

```kira
std.log.info("request_complete", {
    request_id: req.id,
    duration_ms: elapsed,
    status: response.status,
    path: req.path
})
```

## Error Logging

Always log errors with context:

```kira
effect fn process_with_logging(input: Input) -> IO[Result[Output, Error]] {
    let result: Result[Output, Error] = process(input)
    match result {
        Ok(v) => { return Ok(v) }
        Err(e) => {
            std.log.error("Operation failed: input={input}, error={e}")
            return Err(e)
        }
    }
}
```

## Log Correlation

Include identifiers for tracing:

```kira
effect fn handle_request(req: Request) -> IO[Response] {
    let id: string = req.id
    std.log.info("Request started: id={id}")
    // ... processing ...
    std.log.info("Request complete: id={id}, status={status}")
    return response
}
```

## Logging Checklist

- [ ] Logging only in effect functions
- [ ] Context included in messages
- [ ] No secrets or PII in logs
- [ ] No logging in tight loops
- [ ] Errors logged with context
- [ ] Request IDs for correlation
