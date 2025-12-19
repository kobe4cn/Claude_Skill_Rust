---
description: >-
  Use when implementing error handling in Rust. Provides thiserror for libraries
  and anyhow for binaries with proper context and error chaining.
  Triggers: error handling, thiserror, anyhow, Result, error types.
---

# Rust Error Handling Patterns

## Quick Reference

This skill provides guidance for error handling:
- thiserror for library crates (structured errors)
- anyhow for binary crates (rich context)
- Error construction patterns
- Error propagation with context

## Key Principles

### Error Strategy Selection
- **Library crates**: Use `thiserror` for structured error types
- **Binary crates**: Use `anyhow` for simple error handling with context
- **Never use**: `unwrap()` or `expect()` in production code

## Library Crate Errors (thiserror)

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyLibError {
    #[error("Invalid input: {message}")]
    InvalidInput { message: String },

    #[error("File not found: {path}")]
    FileNotFound { path: String },

    #[error("Network error")]
    Network(#[from] reqwest::Error),

    #[error("IO error")]
    Io(#[from] std::io::Error),
}

pub type Result<T> = std::result::Result<T, MyLibError>;

impl MyLibError {
    pub fn invalid_input(msg: impl Into<String>) -> Self {
        Self::InvalidInput { message: msg.into() }
    }
}
```

## Binary Crate Errors (anyhow)

```rust
use anyhow::{Context, Result, bail, ensure};

fn main() -> Result<()> {
    let config = load_config()
        .context("Failed to load configuration")?;

    process_data(&config)
        .context("Failed to process data")?;

    Ok(())
}

fn load_config() -> Result<Config> {
    let path = std::env::var("CONFIG_PATH")
        .context("CONFIG_PATH not set")?;

    let content = std::fs::read_to_string(&path)
        .with_context(|| format!("Failed to read: {}", path))?;

    Ok(toml::from_str(&content)?)
}

fn process_data(config: &Config) -> Result<()> {
    ensure!(!config.files.is_empty(), "No input files");

    if config.strict_mode {
        bail!("Strict mode not supported");
    }

    Ok(())
}
```

## Error Context Patterns

```rust
// Pattern 1: Static context
let data = load_file(path)
    .context("Failed to load configuration")?;

// Pattern 2: Dynamic context
let data = load_file(path)
    .with_context(|| format!("Failed to load: {}", path))?;

// Pattern 3: Error mapping
external_api::call()
    .map_err(|e| anyhow::anyhow!("API failed: {}", e))?;
```

## Error Handling Checklist

```markdown
### Library Crates
- [ ] Errors defined in centralized errors.rs
- [ ] Error types derive Error, Debug
- [ ] Meaningful messages with #[error]
- [ ] #[from] for automatic conversions
- [ ] Helper constructors for errors

### Binary Crates
- [ ] anyhow::Result used consistently
- [ ] Context added to all error chains
- [ ] User-friendly error messages
- [ ] Proper cleanup on fatal errors
```

## Anti-Patterns to Avoid

```rust
// Using unwrap in production
let value = operation().unwrap();

// Ignoring errors
let _ = might_fail();

// Generic error messages
return Err("Something went wrong".into());

// No context
let data = load_file(path)?; // Which file?
```

## See Also

- `reference.md` - Complete patterns and error categories
