---
description: >-
  Use when building CLI applications in Rust. Provides Clap 4.0+ patterns
  including subcommands, enum_dispatch, CommandExecutor trait, and testing.
  Triggers: Clap, CLI, command-line, subcommands, terminal application.
---

# Rust CLI Application Patterns

## Quick Reference

This skill provides guidance for building CLI applications:
- Clap 4.0+ with derive features
- Subcommand architecture with enum_dispatch
- CommandExecutor trait pattern
- Configuration and error handling
- CLI testing with assert_cmd

## Key Requirements

### Clap 4.0+ Standards
- Use derive features for declarative CLI definition
- Implement subcommand architecture for complex CLIs
- Use enum_dispatch for efficient command routing
- Comprehensive error handling with exit codes

## Technology Stack

```toml
[dependencies]
clap = { version = "4.0", features = ["derive", "env", "unicode", "wrap_help"] }
enum_dispatch = "0.3"
anyhow = "1.0"
tokio = { version = "1.45", features = ["macros", "rt-multi-thread"] }
colored = "2.0"
indicatif = "0.17"
dialoguer = "0.11"

[dev-dependencies]
assert_cmd = "2.0"
predicates = "3.0"
```

## CLI Arguments Definition

```rust
#[derive(Debug, Parser)]
#[command(name = "mycli")]
#[command(version = env!("CARGO_PKG_VERSION"))]
#[command(about = "A CLI tool with multiple commands")]
pub struct Args {
    #[arg(short = 'c', long = "config")]
    pub config: Option<PathBuf>,

    #[arg(short = 'v', long = "verbose", action = clap::ArgAction::Count)]
    pub verbose: u8,

    #[command(subcommand)]
    pub command: Commands,
}
```

## CommandExecutor Pattern

```rust
use enum_dispatch::enum_dispatch;

#[async_trait]
#[enum_dispatch(Commands)]
pub trait CommandExecutor {
    async fn execute(&self, args: &Args, config: &Config) -> Result<()>;
}

#[derive(Debug, Subcommand)]
#[enum_dispatch(CommandExecutor)]
pub enum Commands {
    #[command(name = "db")]
    Database(DatabaseCommand),

    #[command(name = "server")]
    Server(ServerCommand),
}
```

## Subcommand Implementation

```rust
#[derive(Debug, Args)]
pub struct DatabaseCommand {
    #[command(subcommand)]
    pub action: DatabaseAction,
}

#[derive(Debug, Subcommand)]
pub enum DatabaseAction {
    Init {
        #[arg(long, env = "DATABASE_URL")]
        url: Option<String>,
        #[arg(long, short = 'f')]
        force: bool,
    },
    Status {
        #[arg(long)]
        detailed: bool,
    },
}

#[async_trait]
impl CommandExecutor for DatabaseCommand {
    async fn execute(&self, args: &Args, config: &Config) -> Result<()> {
        match &self.action {
            DatabaseAction::Init { url, force } => {
                // Implementation
            }
            DatabaseAction::Status { detailed } => {
                // Implementation
            }
        }
        Ok(())
    }
}
```

## User Feedback Patterns

```rust
use colored::*;
use indicatif::{ProgressBar, ProgressStyle};

// Colored output
println!("{}", "Success!".green());
println!("{}", "Warning: ...".yellow());
println!("{}", "Error: ...".red());

// Progress bar
let pb = ProgressBar::new(100);
pb.set_style(ProgressStyle::default_bar()
    .template("{spinner:.green} [{bar:40}] {pos}/{len}")
    .unwrap());

// Interactive confirmation
use dialoguer::Confirm;
if Confirm::new().with_prompt("Continue?").interact()? {
    // Proceed
}
```

## CLI Testing

```rust
use assert_cmd::Command;
use predicates::prelude::*;

#[test]
fn test_help_command() {
    Command::cargo_bin("mycli").unwrap()
        .arg("--help")
        .assert()
        .success()
        .stdout(predicate::str::contains("A CLI tool"));
}

#[test]
fn test_database_status() {
    Command::cargo_bin("mycli").unwrap()
        .args(["db", "status"])
        .assert()
        .success();
}
```

## Anti-Patterns to Avoid

- Not using derive features
- Missing enum_dispatch for command routing
- Blocking operations without progress indication
- Missing error context and exit codes
- No interactive confirmation for destructive actions

## See Also

- `reference.md` - Complete patterns, configuration, and error handling
