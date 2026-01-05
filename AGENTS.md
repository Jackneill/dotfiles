# General guidelines

This document captures code conventions for the projects in this folder.
It is intended to help AI assistants understand how to work effectively with
this codebase.

## SPZ File Format

For the `.SPZ` file format specification and details see `./docs/SPZ.md`.

## General conventions

### Correctness over convenience

- Model the full error space—no shortcuts or simplified error handling.
- Handle all edge cases, including race conditions, signal timing, and platform differences.
- Use the type system to encode correctness constraints.
- Prefer compile-time guarantees over runtime checks where possible.

### User experience as a primary driver

- Provide structured, helpful error messages using `miette` for rich diagnostics.
- Make progress reporting responsive and informative.
- Maintain consistency across platforms even when underlying OS capabilities differ.
 Use OS-native logic rather than trying to emulate Unix on Windows or vice versa.
- Write user-facing messages in clear, present tense: "spz now supports..." not "spz now supported..."

### Pragmatic incrementalism

- "Not overly generic"—prefer specific, composable logic over abstract frameworks.
- Evolve the design incrementally rather than attempting perfect upfront architecture.

### Production-grade engineering

- Use type system extensively: newtypes, builder patterns, type states, lifetimes.
- Use message passing or the actor model to avoid data races.
- Comprehensive testing including edge cases, race conditions, and stress tests.
- Getting the details right is really important!

### Documentation

- Use inline comments to explain "why," not just "what".
- Module-level documentation should explain purpose and responsibilities.

## Code style

### File headers

Every Rust source file must start with:

```rust
// SPDX-License-Identifier: MIT OR Apache-2.0
```

### Rust edition and formatting

- Use Rust 2024 edition.

### Type system patterns

- **Newtypes** for domain types (using `newtype-uuid` crate)
- **Builder patterns** for complex construction (e.g., `TestRunnerBuilder`)
- **Type states** encoded in generics when state transitions matter
- **Lifetimes** used extensively to avoid cloning (e.g., `TestInstance<'a>`)
- **Restricted visibility**: Use `pub(crate)` and `pub(super)` liberally
- **Non-exhaustive**: All public error types should be `#[non_exhaustive]` for forward compatibility

### Error handling

- Use `thiserror` for error types with `#[derive(Error)]`.
- Group errors by category with an `ErrorKind` enum when appropriate.
- Provide rich error context using `miette` for user-facing errors.
- Two-tier error model:
  - `ExpectedError`: User/external errors with semantic exit codes.
  - Internal errors: Programming errors that may panic or use internal error types.
- Error display messages should be lowercase sentence fragments suitable for "failed to {error}".

### Async patterns

- Use `tokio` for async runtime (multi-threaded).
- Be selective with async. Only use it in runner and runner-adjacent code.
- Use async for I/O and concurrency, keep other code synchronous.
- Use `async-scoped` for structured concurrency without `'static` bounds.
- Use `future-queue` for backpressure-aware task scheduling.
- Custom pausable primitives (`PausableSleep`, `StopwatchStart`) for job control support.

### Module organization

- Use `mod.rs` files to re-export public items.
- Do not put any nontrivial logic in `mod.rs`.
- Keep module boundaries strict with restricted visibility.
- Platform-specific code in separate files: `unix.rs`, `windows.rs`.
- Use `#[cfg(unix)]` and `#[cfg(windows)]` for conditional compilation.
- Test helpers in dedicated modules/files.

### Memory and performance

- Use `Arc` or borrows for shared immutable data.
- Use `smol_str` for efficient small string storage.
- Careful attention to copying vs. referencing.
- Use `debug-ignore` to avoid expensive debug formatting in hot paths.
- Stream data where possible rather than buffering.

### Syntax

- Always use an underscore when suffixing an integer with it's type: example:
  - DO NOT use `0u32`
  - DO USE `0_u32`

## Testing Practices

### Running Tests

**CRITICAL**: Always use `cargo nextest run` to run unit and integration tests.
 Never use `cargo test` for these!
 For doctests, use `cargo test --doc`.

### Test Organization

- Unit tests in the same file as the code they test.
- Integration tests in `tests/integration` crate.
- Test utilities in `tests/utilities` crate.

### Testing Tools

- **test-case**: For parameterized tests.
- **proptest**: For property-based testing.
- **insta**: For snapshot testing.
- **libtest-mimic**: For custom test harnesses.
- **pretty_assertions**: For better assertion output.

## Commit message style

### Format

Commits follow a conventional format with crate-specific scoping:

```
[crate-name] brief description (#PR-number)
```

Examples:

- `[nextest-runner] add --max-progress-running, cap to 8 by default (#2727)`
- `[cargo-nextest] version 0.9.111`
- `[meta] update MSRV to Rust 1.88 (#2725)`

### Conventions

- Use `[meta]` for cross-cutting concerns (MSRV updates, releases, CI changes)
- Version bump commits: `[crate-name] version X.Y.Z` (these are performed by `cargo release`)
- Release preparation: `[meta] prepare releases`
- Include PR number for all non-version commits
- Use lowercase for descriptions
- Keep descriptions concise but descriptive

### Commit quality

- **Atomic commits**: Each commit should be a logical unit of change
- **Bisect-able history**: Every commit must build and pass all checks
- **Separate concerns**: Don't mix formatting fixes with logic changes
- Format fixes and refactoring should be in separate commits from feature changes

## Architecture

### Key Design Principles

1. **No direct state sharing**—everything via message passing
2. **Linearized events**—dispatcher ensures consistent view
3. **Full error space modeling**—handle all failure modes
4. **Pausable timers**—custom implementations for job control (SIGTSTP/SIGCONT)
5. **Structured concurrency**—use `async-scoped` for spawning with borrows

### Cross-Platform Strategy

- Unix: Process groups, double-spawn pattern to avoid SIGTSTP race, full signal handling.
- Windows: Job objects, console mode manipulation, limited signal support.
- Conditional compilation: `#[cfg(unix)]`, `#[cfg(windows)]`.
- Platform modules: `unix.rs`, `windows.rs` with shared interfaces.
- Document platform differences and trade-offs in code comments.

## Crates

- `spz` - spz file format handling for rust and cli tools.

## Dependencies

### Workspace Dependencies

- All versions managed in root `Cargo.toml` `[workspace.dependencies]`
- Internal crates use exact version pinning: `version = "=0.17.0"`
- Comment on dependency choices when non-obvious
- Example: "Disable punycode parsing since we only access well-known domains"

### Key Dependencies

- **tokio**: Async runtime, essential for concurrency model
- **anyhow**: Error helper crate
- **thiserror**: Error derive macros
- **serde**: Serialization (config, metadata)
- **clap**: CLI parsing with derives

## Quick Reference

### Commands

```bash
# Run tests (ALWAYS use nextest for unit/integration tests)
just test

# Run doctests (nextest doesn't support these)
cargo test --doc

# Lint
just lint

# Build
just build

# Release
just build-release
```

### Helpful Git Commands

```bash
# Get commits since last release
git log <previous-tag>..main --oneline

# Check if contributor is first-time
git log --all --author="Name" --oneline | wc -l

# Get PR author username
gh pr view <number> --json author --jq '.author.login'

# View commit details
git show <commit> --stat
```
