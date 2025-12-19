# Rust Simple Crate Reference

## Binary Crate Complete Structure

```
my_project/
├── Cargo.toml
├── README.md
├── src/
│   ├── main.rs           # Entry point (minimal)
│   ├── lib.rs            # Core application logic
│   ├── errors.rs         # Centralized error definitions
│   ├── config.rs         # Configuration handling
│   ├── cli.rs            # Command-line interface
│   └── modules/          # Feature-based modules
│       ├── mod.rs        # Module declarations
│       ├── auth.rs       # Authentication logic
│       ├── database.rs   # Database operations
│       └── handlers.rs   # Request/command handlers
├── tests/                # Integration tests
│   └── integration_test.rs
└── examples/             # Usage examples
    └── basic_usage.rs
```

## Minimal main.rs

```rust
// src/main.rs - Keep this minimal and delegate to lib.rs
use anyhow::Result;

fn main() -> Result<()> {
    my_project::run()
}
```

## Comprehensive lib.rs for Binary

```rust
// src/lib.rs - Contains the main application logic
mod config;
mod errors;
mod cli;
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

/// Core application struct
pub struct Application {
    config: Config,
}

impl Application {
    pub fn new(config: Config) -> Result<Self> {
        Ok(Self { config })
    }

    pub fn start(&self) -> Result<()> {
        println!("Application started with config: {:?}", self.config);
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

## Configuration Module

```rust
// src/config.rs
use serde::{Deserialize, Serialize};
use std::env;
use anyhow::{Context, Result};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Config {
    pub database_url: String,
    pub server_port: u16,
    pub log_level: String,
    pub debug_mode: bool,
}

impl Config {
    pub fn load() -> Result<Self> {
        if let Ok(config) = Self::from_env() {
            return Ok(config);
        }

        Self::from_file("config.toml")
            .context("Failed to load configuration from file")
    }

    fn from_env() -> Result<Self> {
        Ok(Self {
            database_url: env::var("DATABASE_URL")
                .context("DATABASE_URL not set")?,
            server_port: env::var("SERVER_PORT")
                .unwrap_or_else(|_| "8080".to_string())
                .parse()
                .context("Invalid SERVER_PORT")?,
            log_level: env::var("LOG_LEVEL")
                .unwrap_or_else(|_| "info".to_string()),
            debug_mode: env::var("DEBUG_MODE")
                .map(|v| v.parse().unwrap_or(false))
                .unwrap_or(false),
        })
    }

    fn from_file(path: &str) -> Result<Self> {
        let content = std::fs::read_to_string(path)
            .with_context(|| format!("Failed to read: {}", path))?;
        toml::from_str(&content).context("Failed to parse config")
    }
}

impl Default for Config {
    fn default() -> Self {
        Self {
            database_url: "sqlite::memory:".to_string(),
            server_port: 8080,
            log_level: "info".to_string(),
            debug_mode: false,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_default_config() {
        let config = Config::default();
        assert_eq!(config.server_port, 8080);
        assert!(!config.debug_mode);
    }
}
```

## Library Crate Structure

```
my_lib/
├── Cargo.toml
├── README.md
├── src/
│   ├── lib.rs            # Public API and module declarations
│   ├── errors.rs         # Error types (using thiserror)
│   └── modules/          # Functionality modules
│       ├── mod.rs        # Module re-exports
│       ├── core.rs       # Core functionality
│       ├── utils.rs      # Utility functions
│       └── types.rs      # Public type definitions
├── tests/
│   └── lib_test.rs
├── examples/
│   └── quick_start.rs
└── benches/
    └── benchmark.rs
```

## Library Public API

```rust
// src/lib.rs - Clean public API
//! # My Library
//!
//! This library provides functionality for...
//!
//! ## Quick Start
//!
//! ```rust
//! use my_lib::MyStruct;
//!
//! let instance = MyStruct::new("example")?;
//! let result = instance.process()?;
//! ```

mod modules;
mod errors;

// Public API exports
pub use errors::{MyLibError, Result};
pub use modules::{
    core::{MyStruct, ProcessResult},
    utils::{helper_function, UtilityTrait},
};
pub use modules::types::{PublicType, Configuration};

pub const VERSION: &str = env!("CARGO_PKG_VERSION");

pub fn init() -> Result<()> {
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_library_initialization() {
        assert!(init().is_ok());
    }
}
```

## Binary Cargo.toml

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2024"
authors = ["Your Name <email@example.com>"]
description = "A brief description"
readme = "README.md"
license = "MIT OR Apache-2.0"

[dependencies]
anyhow = "1.0"
serde = { version = "1.0", features = ["derive"] }
toml = "0.8"
clap = { version = "4.0", features = ["derive"] }

[dev-dependencies]
tempfile = "3.0"

[[bin]]
name = "my_project"
path = "src/main.rs"

[profile.release]
lto = true
codegen-units = 1
panic = "abort"
```

## Library Cargo.toml

```toml
[package]
name = "my_lib"
version = "0.1.0"
edition = "2024"
authors = ["Your Name <email@example.com>"]
description = "A useful library for..."
readme = "README.md"
license = "MIT OR Apache-2.0"
keywords = ["library", "utility"]
categories = ["development-tools"]

[dependencies]
thiserror = "2.0"
serde = { version = "1.0", features = ["derive"] }

[dev-dependencies]
serde_json = "1.0"

[lib]
name = "my_lib"
path = "src/lib.rs"

[features]
default = []
extra = ["serde"]

[[example]]
name = "quick_start"
path = "examples/quick_start.rs"
```

## Module Organization

```rust
// src/modules/mod.rs
pub mod core;
pub mod utils;
pub mod types;

// Re-export for internal use
pub(crate) use core::*;
pub(crate) use utils::*;
```

```rust
// src/modules/core.rs
use crate::errors::{MyLibError, Result};

#[derive(Debug, Clone)]
pub struct MyStruct {
    data: String,
}

impl MyStruct {
    pub fn new(data: impl Into<String>) -> Result<Self> {
        let data = data.into();
        if data.is_empty() {
            return Err(MyLibError::invalid_input("Data cannot be empty"));
        }
        Ok(Self { data })
    }

    pub fn process(&self) -> Result<ProcessResult> {
        let processed = self.data.to_uppercase();
        Ok(ProcessResult {
            original: self.data.clone(),
            processed,
        })
    }
}

#[derive(Debug, Clone, PartialEq)]
pub struct ProcessResult {
    pub original: String,
    pub processed: String,
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_mystruct_creation() {
        let instance = MyStruct::new("test").unwrap();
        assert_eq!(instance.data, "test");
    }

    #[test]
    fn test_empty_data_error() {
        let result = MyStruct::new("");
        assert!(result.is_err());
    }

    #[test]
    fn test_processing() {
        let instance = MyStruct::new("hello").unwrap();
        let result = instance.process().unwrap();
        assert_eq!(result.processed, "HELLO");
    }
}
```

## Single Crate Checklist

```markdown
### Single Crate Structure Verification

#### Project Setup
- [ ] Appropriate crate type chosen (bin/lib/mixed)
- [ ] Cargo.toml properly configured
- [ ] README.md with clear documentation
- [ ] License file included

#### File Organization
- [ ] src/main.rs minimal (for binary crates)
- [ ] src/lib.rs contains core logic
- [ ] src/errors.rs centralizes error definitions
- [ ] Feature modules in src/modules/
- [ ] Each file ≤ 500 lines (excluding tests)

#### Code Quality
- [ ] Functions ≤ 150 lines
- [ ] Functionality-based file organization
- [ ] Comprehensive unit tests in each file
- [ ] Public API well-documented

#### Testing
- [ ] Unit tests in each module
- [ ] Integration tests in tests/ directory
- [ ] Examples in examples/ directory
```

---
*Source: .cursor/rules/rust/simple/single-crate.mdc*
