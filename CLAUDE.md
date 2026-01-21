## Kira Development

This project uses the Kira-Toolkit framework (v0.1.0) for Kira development standards.

See `kira-toolkit/KIRATOOLKIT.md` for coding guidelines and available commands.

See `kira-toolkit/STANDARDS.md` for full coding standards and language specification.

### Key Syntax Requirements
- **Explicit types**: `let x: i32 = 42`
- **Explicit return**: `return value` (not implicit last expression)
- **Pure by default**: Use `effect fn` for IO operations
- **Pattern matching**: All matches must be exhaustive
- **Closures with full types**: `fn(x: i32) -> i32 { return x * 2 }`
