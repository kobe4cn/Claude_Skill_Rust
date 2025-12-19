---
description: >-
  Use when organizing simple single-crate Rust projects. Covers minimal main.rs pattern,
  binary vs library vs mixed crate types, configuration module patterns, and
  file organization best practices.
  Triggers: single crate, binary crate, library crate, minimal main, project structure.
---

# Rust Simple Crate Structure

## Quick Reference

Guidelines for organizing simple Rust projects using a single crate structure.

## Binary Crate Structure

```
my_project/
├── Cargo.toml
├── src/
│   ├── main.rs           # Minimal entry point
│   ├── lib.rs            # Core application logic
│   ├── errors.rs         # Centralized error definitions
│   ├── config.rs         # Configuration handling
│   └── modules/          # Feature-based modules
│       ├── mod.rs
│       ├── auth.rs
│       └── handlers.rs
├── tests/
│   └── integration_test.rs
└── examples/
    └── basic_usage.rs
```

## Minimal main.rs Pattern

```rust
// src/main.rs - Keep this minimal
use anyhow::Result;

fn main() -> Result<()> {
    my_project::run()
}
```

## Comprehensive lib.rs

```rust
// src/lib.rs - Core application logic
mod config;
mod errors;
mod modules;

pub use errors::AppError;
use config::Config;
use anyhow::{Context, Result};

/// Main application entry point
pub fn run() -> Result<()> {
    let config = Config::load()
        .context("Failed to load configuration")?;

    let app = Application::new(config)?;
    app.start()
        .context("Application failed to start")?;

    Ok(())
}

pub struct Application {
    config: Config,
}

impl Application {
    pub fn new(config: Config) -> Result<Self> {
        Ok(Self { config })
    }

    pub fn start(&self) -> Result<()> {
        println!("Starting with config: {:?}", self.config);
        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_application_creation() {
        let config = Config::default();
        let app = Application::new(config);
        assert!(app.is_ok());
    }
}
```

## Library Crate Structure

> **Tip**: For library projects with CI/CD, pre-commit hooks, and security tooling pre-configured,
> consider using the lib-template: `cargo generate --git https://github.com/kobe4cn/rust-lib-template`
> See `../lib-template/` for details.

```
my_lib/
├── Cargo.toml
├── src/
│   ├── lib.rs            # Public API
│   ├── errors.rs         # Error types (thiserror)
│   └── modules/
│       ├── mod.rs
│       ├── core.rs
│       └── utils.rs
├── tests/
│   └── lib_test.rs
└── examples/
    └── quick_start.rs
```

## Library lib.rs Pattern

```rust
// src/lib.rs - Clean public API
//! # My Library
//!
//! ```rust
//! use my_lib::MyStruct;
//!
//! let instance = MyStruct::new("example")?;
//! let result = instance.process()?;
//! ```

mod modules;
mod errors;

pub use errors::{MyLibError, Result};
pub use modules::core::{MyStruct, ProcessResult};

pub const VERSION: &str = env!("CARGO_PKG_VERSION");

pub fn init() -> Result<()> {
    Ok(())
}
```

## Anti-Patterns

```rust
// ❌ Fat main.rs with business logic
fn main() {
    // Hundreds of lines of business logic
    let config = load_config();
    // ... more logic
}

// ❌ Type-based file organization
// src/types.rs - All types mixed together
// src/traits.rs - All traits mixed together

// ✅ Do This Instead
// Minimal main.rs
fn main() -> anyhow::Result<()> {
    my_project::run()
}

// Function-based organization
// src/auth.rs - Authentication-related types and logic
// src/database.rs - Database-related functionality
```

## See Also

- `reference.md` - Complete patterns and Cargo.toml templates
- `../lib-template/` - Standardized library project template with CI/CD, pre-commit hooks, and security tooling
- `../workspace/` - Multi-crate workspace organization
- `../core/` - Code quality standards
