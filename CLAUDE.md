# Agent Instructions for Rust Project

This repository is a strict, general-purpose single-crate Rust project. Adherence to established conventions and code quality standards is mandatory.

## 1. Operational Commands

### Task Runner (`Justfile`)

The primary interface for development tasks is `just`.

- **Lint**: `just lint` (Runs clippy with strict settings)
- **Test**: `just test` (Runs all tests)
- **Check**: `just check` (Syntax check)
- **Format**: `just fmt` (Applies strict formatting)
- **Fix**: `just fix` (Auto-fixes clippy issues where possible)

### Cargo Commands (Manual)

If `just` is unavailable, use standard cargo commands with these specific flags:

- **Lint**: `cargo clippy --all-targets --all-features -- -D warnings`
- **Test**: `cargo test --all-targets --all-features`
- **Build**: `cargo build`

### Running a Single Test

To run a specific test case:

```bash
cargo test <test_function_name> -- --nocapture
```

Example: `cargo test verify_initialization -- --nocapture`

### Continuous Development (`bacon`)

Use `bacon` for a background compile-check loop:

- `bacon` (Defaults to `run` job)
- `bacon clippy`
- `bacon test`

## 2. Code Style & Standards

### General Philosophy

Prioritize code correctness and clarity. Speed and efficiency are secondary priorities unless otherwise specified.

### Comments & Documentation

- **Philosophy**: "Every comment represents a failure to express ourself in code."
- **Hierarchy of Clarity**:
  1.  **Refactor**: Prioritize renaming or restructuring code to be self-describing.
  2.  **Explain "Why"**: If code cannot be simplified further, comment on _why_ it is written that way (e.g., specific constraints, edge cases).
  3.  **Explain "What"**: Only comment on _what_ code does if the complexity is essential and cannot be reduced, and the logic remains non-obvious.
- **Avoid**: Do not write organizational comments or summaries of obvious logic.

### Strictness & Lints

This project enforces strict code quality via `Cargo.toml`.

- **Unsafe Code**: Forbidden (`unsafe_code = "forbid"`).
- **Warnings**: Denied (`warnings = "deny"`).
- **Clippy**:
  - `clippy::all`, `clippy::pedantic`, `clippy::nursery` are enabled at `warn` level (and thus denied by `-D warnings` in CI/Justfile).
  - **Explicitly Denied**: `unwrap_used`, `panic`, `todo`, `unimplemented`.
  - **Action**: Never use `.unwrap()`. Use `.expect("context")` ONLY if mathematically impossible to fail (and document why), otherwise handle the error. Prefer `?` propagation.

### Formatting (`rustfmt.toml`)

Formatting is enforced automatically. Do not manually align imports or variables if `rustfmt` reverts it.

- **Edition**: 2024
- **Imports**: `imports_granularity = "Crate"`, `group_imports = "StdExternalCrate"`.
- **Shorthand**: Use field init shorthand (`Point { x, y }` instead of `Point { x: x, y: y }`).

### Error Handling

- Use `thiserror` to derive custom error types for domain-specific errors.
- Use `anyhow::Result` for top-level and application error propagation.
- **Panic**: Do not panic. Return `Result` for all fallible operations.
- **Indexing**: Be careful with operations like indexing which may panic if the indexes are out of bounds. Prefer `.get()` or ensure bounds are checked.

### Logging & Observability

- Use the `tracing` crate, not `println!`.
- **Debug**: `tracing::debug!(?var, "message")`
- **Info**: `tracing::info!("Startup complete")`
- **Error**: `tracing::error!(%err, "Operation failed")`

### Async Contexts

- Use variable shadowing to scope clones in async contexts for clarity, minimizing the lifetime of borrowed references.
  Example:
  ```rust
  tokio::spawn({
      let state = state.clone();
      async move {
          state.process().await;
      }
  });
  ```

### Project Structure

This is a **single-crate Rust project** (not a workspace).

- All source code lives in `src/`.
- **Modules**: Never create files with `mod.rs` paths. Prefer `src/some_module.rs` instead of `src/some_module/mod.rs` to ensure a consistent and modern file structure.
- Organize modules by domain/feature, not by technical layer.

### Dependencies

- **Safety**: Do not add dependencies that require `unsafe` unless absolutely necessary and justified.
- **Minimalism**: Prefer existing dependencies before adding new ones.

### Type-Driven Design

Leverage Rust's type system to encode domain invariants at compile time. Invalid states should be unrepresentable.

- **Prefer Value Objects**: Wrap primitives in newtypes that enforce invariants at construction (`struct Email(String)`, `struct UserId(Uuid)`).
- **Use Enums for Variants**: Model mutually exclusive states as enum variants rather than boolean flags or string codes.
- **Typestate Pattern**: Encode state machines in the type system so invalid transitions fail to compile.

**Typestate Example**: Instead of runtime checks for operation ordering, use separate types per state:

```rust
struct HttpResponse<S: ResponseState> {
    inner: Box<ResponseData>,
    _state: PhantomData<S>,
}

struct Start;
struct HeadersSent;

impl HttpResponse<Start> {
    fn status_line(self, code: u16) -> HttpResponse<HeadersSent> { ... }
}

impl HttpResponse<HeadersSent> {
    fn header(&mut self, key: &str, value: &str) { ... }
    fn body(self, content: &str) { ... }
}
```

With this pattern:
- Calling `header()` before `status_line()` is a compile error (method doesn't exist on `HttpResponse<Start>`)
- Calling `status_line()` twice is a compile error (first call consumes `self`)
- After `body()` consumes the response, no further operations are possible

**When to Use Typestate**:
- Protocol implementations with ordered phases (handshakes, request/response cycles)
- Builder patterns where certain fields must be set before others
- Resource lifecycle management (open -> use -> close)
- Workflow engines with defined state transitions

**Benefits**:
- Compile-time enforcement of business rules
- Self-documenting APIs where valid operations are discoverable via IDE
- Zero runtime overhead (states are phantom types)
- Eliminates entire classes of "should never happen" bugs

## 3. Workflow for Agents

1.  **Plan**: Analyze requirements, module locations, and existing patterns.
2.  **Edit**: Apply changes. Ensure you add/update tests.
3.  **Verify**:
    - Run `just fmt` to fix style.
    - Run `just lint` to ensure no strict rules are violated (no unwraps!).
    - Run `just test` to verify logic.
4.  **Commit**: Ensure the codebase is green before declaring a task complete.
