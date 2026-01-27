# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Starship is a minimal, blazing-fast, infinitely customizable cross-shell prompt written in Rust. It uses parallel module execution (rayon) and supports 100+ configurable modules for displaying contextual information (git status, language versions, AWS regions, etc.).

## Common Commands

### Build
```bash
cargo build --release    # Release build
cargo build              # Dev build (faster)
cargo check              # Quick compilation check
```

### Test
```bash
cargo test                              # All tests
cargo test test_name                    # Single test
cargo test --package starship --lib modules::python  # Module tests
```

### Lint and Format (must pass CI)
```bash
cargo fmt                               # Format code
cargo fmt --all -- --check              # Check formatting
cargo clippy --all-targets --all-features  # Lint

# Markdown/TOML formatting
cargo install dprint
dprint fmt
```

### Run
```bash
cargo run -- prompt        # Run starship prompt
STARSHIP_LOG=trace cargo run -- prompt  # Debug logging
```

### Config Schema (update when config changes)
```bash
cargo run --features config-schema -- config-schema > .github/config-schema.json
```

## Architecture

### Module-Based Parallel Execution
- Entry point: `src/main.rs` (CLI via clap)
- Prompt orchestration: `src/print.rs` (parallel module execution with rayon)
- Context: `src/context.rs` (shared state: env, git, dirs, config)
- Module trait: `src/module.rs` with `ALL_MODULES` registry
- Formatter: `src/formatter/` (StringFormatter with variable interpolation)

### Adding a New Module
1. Config struct in `src/configs/<module>.rs`
2. Add to `PROMPT_ORDER` in `src/configs/starship_root.rs`
3. Add to `FullConfig` in `src/configs/mod.rs`
4. Add to `ALL_MODULES` in `src/module.rs`
5. Add `mod` declaration in `src/modules/mod.rs`
6. Add entry in `handle()` function in `src/modules/mod.rs`
7. Implement in `src/modules/<module>.rs`
8. Add docs to `docs/config/README.md`
9. Update presets in `docs/public/presets/toml/`
10. Update config schema (see command above)

### Testing Framework
- Use `ModuleRenderer` in `src/test/mod.rs` for module output testing
- Mock environment via `.env()`, `.path()`, `.config()`, `.cmd()`
- Use `fixture_repo()` for VCS testing (call `.close()` when done)
- File I/O tests must use `sync_all()` when creating/writing files
- Tests requiring external tools: mark `#[ignored]` (run in CI only)

## Code Conventions

### Environment Variables
```rust
// Correct (mockable)
context.get_env("VAR_NAME")?

// Wrong (not mockable)
std::env::var("VAR_NAME")
```

### External Commands
```rust
// Correct (mockable)
context.exec_cmd("cmd", &["arg1", "arg2"])?.stdout

// If context.exec_cmd not possible, use:
crate::utils::create_command("cmd")  // NOT std::process::Command::new
```

### Absolute Paths
```rust
// Correct (mockable in tests)
crate::utils::context_path(context, "/absolute/path")

// Wrong (breaks tests)
std::fs::canonicalize()
PathBuf::from("/absolute/path")
```

### Clippy Restrictions
- Disallowed: `std::process::Command::new`, `std::env::set_var`, `std::fs::canonicalize`
- Use alternatives: `crate::utils::create_command`, context env methods, `dunce::canonicalize`

### External Command Mocking
When adding mocked command output in `src/utils.rs`, match arguments exactly as called:
```rust
utils::exec_cmd("program", &["arg", "more_args"])  // Matches "program arg more_args"
```
