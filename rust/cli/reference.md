# Rust CLI Reference

## Complete CLI Structure

```rust
// src/main.rs
use anyhow::Result;

mod cli;
mod commands;
mod config;

use cli::Args;
use commands::CommandExecutor;

#[tokio::main]
async fn main() -> Result<()> {
    let args = Args::parse();
    let config = config::Config::load(&args.config)?;

    match args.command.execute(&args, &config).await {
        Ok(()) => Ok(()),
        Err(e) => {
            eprintln!("Error: {}", e);
            std::process::exit(1);
        }
    }
}
```

## Complete Arguments Definition

```rust
// src/cli.rs
use clap::{Parser, Subcommand, ValueEnum};
use std::path::PathBuf;

#[derive(Debug, Parser)]
#[command(name = "mycli")]
#[command(version = env!("CARGO_PKG_VERSION"))]
#[command(about = "A CLI tool with multiple commands")]
#[command(arg_required_else_help = true)]
pub struct Args {
    #[arg(short = 'c', long = "config", value_name = "FILE")]
    pub config: Option<PathBuf>,

    #[arg(short = 'v', long = "verbose", action = clap::ArgAction::Count)]
    pub verbose: u8,

    #[arg(long = "format", value_enum, default_value = "text")]
    pub format: OutputFormat,

    #[arg(long = "no-color")]
    pub no_color: bool,

    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Debug, Clone, ValueEnum)]
pub enum OutputFormat {
    Text,
    Json,
    Yaml,
}
```

## Complete Command Implementation

```rust
// src/commands/mod.rs
use anyhow::Result;
use async_trait::async_trait;
use clap::Subcommand;
use enum_dispatch::enum_dispatch;

pub mod database;
pub mod server;

#[async_trait]
#[enum_dispatch(Commands)]
pub trait CommandExecutor {
    async fn execute(&self, args: &Args, config: &Config) -> Result<()>;
}

#[derive(Debug, Subcommand)]
#[enum_dispatch(CommandExecutor)]
pub enum Commands {
    #[command(name = "db", alias = "database")]
    Database(database::DatabaseCommand),

    #[command(name = "server", alias = "srv")]
    Server(server::ServerCommand),
}
```

## Subcommand with Actions

```rust
// src/commands/database.rs
use clap::{Args, Subcommand};
use colored::*;

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
    Backup {
        #[arg(short = 'o', long = "output")]
        output: std::path::PathBuf,
    },
}

#[async_trait]
impl CommandExecutor for DatabaseCommand {
    async fn execute(&self, args: &Args, config: &Config) -> Result<()> {
        match &self.action {
            DatabaseAction::Init { url, force } => {
                if *force {
                    println!("{}", "Force init...".yellow());
                }
                println!("{}", "Database initialized".green());
            }
            DatabaseAction::Status { detailed } => {
                println!("Status: {}", "Connected".green());
            }
            DatabaseAction::Backup { output } => {
                println!("Backup saved to: {}", output.display());
            }
        }
        Ok(())
    }
}
```

## Configuration Management

```rust
// src/config.rs
use anyhow::{Context, Result};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Config {
    pub app: AppConfig,
    pub database: DatabaseConfig,
}

impl Config {
    pub fn load(config_path: &Option<std::path::PathBuf>) -> Result<Self> {
        if let Some(path) = config_path {
            let content = std::fs::read_to_string(path)
                .with_context(|| format!("Failed to read: {}", path.display()))?;

            match path.extension().and_then(|e| e.to_str()) {
                Some("json") => serde_json::from_str(&content)
                    .context("Failed to parse JSON"),
                Some("yaml") | Some("yml") => serde_yaml::from_str(&content)
                    .context("Failed to parse YAML"),
                _ => anyhow::bail!("Unsupported format"),
            }
        } else {
            Ok(Self::default())
        }
    }
}
```

## Error Handling

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum CliError {
    #[error("Configuration error: {0}")]
    Config(#[from] ConfigError),

    #[error("Command failed: {0}")]
    Command(String),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}

impl CliError {
    pub fn exit_code(&self) -> i32 {
        match self {
            CliError::Config(_) => 2,
            CliError::Command(_) => 1,
            CliError::Io(_) => 3,
        }
    }
}
```

## Progress and User Feedback

```rust
use indicatif::{ProgressBar, ProgressStyle};
use dialoguer::Confirm;

// Progress bar
let pb = ProgressBar::new(100);
pb.set_style(ProgressStyle::default_bar()
    .template("{spinner:.green} [{bar:40}] {pos}/{len} {msg}")
    .unwrap());

for i in 0..100 {
    pb.set_position(i + 1);
    pb.set_message(format!("Processing item {}", i));
    tokio::time::sleep(Duration::from_millis(50)).await;
}
pb.finish_with_message("Done");

// Confirmation dialog
if Confirm::new()
    .with_prompt("Continue with this action?")
    .default(false)
    .interact()? {
    // Proceed
}
```

## CLI Testing

```rust
use assert_cmd::Command;
use predicates::prelude::*;

#[test]
fn test_help() {
    Command::cargo_bin("mycli").unwrap()
        .arg("--help")
        .assert()
        .success()
        .stdout(predicate::str::contains("A CLI tool"));
}

#[test]
fn test_version() {
    Command::cargo_bin("mycli").unwrap()
        .arg("--version")
        .assert()
        .success()
        .stdout(predicate::str::contains(env!("CARGO_PKG_VERSION")));
}

#[test]
fn test_db_status() {
    Command::cargo_bin("mycli").unwrap()
        .args(["db", "status"])
        .assert()
        .success();
}

#[test]
fn test_json_output() {
    Command::cargo_bin("mycli").unwrap()
        .args(["--format", "json", "db", "status"])
        .assert()
        .success()
        .stdout(predicate::str::is_json());
}

#[test]
fn test_invalid_subcommand() {
    Command::cargo_bin("mycli").unwrap()
        .arg("invalid")
        .assert()
        .failure();
}
```

## CLI Checklist

```markdown
### Command Structure
- [ ] Uses clap 4.0+ with derive features
- [ ] Subcommand architecture implemented
- [ ] CommandExecutor trait with async support
- [ ] enum_dispatch for efficient routing

### Argument Handling
- [ ] Global arguments (config, verbose, format)
- [ ] Subcommand-specific arguments
- [ ] Environment variable integration
- [ ] Help text for all commands

### Error Handling
- [ ] Structured error types with thiserror
- [ ] User-friendly error messages
- [ ] Appropriate exit codes

### User Experience
- [ ] Colored output
- [ ] Progress bars for long operations
- [ ] Interactive prompts for destructive actions
- [ ] Multiple output formats

### Testing
- [ ] Integration tests with assert_cmd
- [ ] Command output validation
- [ ] Error condition testing
```

---
*Source: cursor-rust-rules features/cli.mdc*
