---
description: >-
  Use when implementing configuration management in Rust. Provides figment patterns
  including multi-format parsing, validation, hot-reloading with arc-swap.
  Triggers: configuration, figment, YAML, TOML, hot-reload, settings.
---

# Rust Configuration Management

## Quick Reference

This skill provides guidance for configuration management:
- Multi-format support (YAML, TOML, JSON) with figment
- Validation with validator crate
- Hot-reloading with arc-swap
- Environment variable overrides

## Key Requirements

### Configuration Standards
- Use figment for multi-format parsing
- Validate all configurations with validator
- Support environment variable overrides
- Use arc-swap for hot-reloadable config

## Technology Stack

```toml
[dependencies]
figment = { version = "0.10", features = ["toml", "yaml", "json", "env"] }
arc-swap = "1.0"
validator = { version = "0.18", features = ["derive"] }
notify = "6.0"
serde = { version = "1.0", features = ["derive"] }
```

## Configuration Structure

```rust
use figment::{Figment, providers::{Format, Yaml, Toml, Json, Env}};
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, Validate)]
#[serde(rename_all = "snake_case")]
pub struct AppConfig {
    #[validate(length(min = 1, max = 100))]
    pub name: String,

    #[validate(range(min = 1, max = 65535))]
    pub port: u16,

    #[serde(default = "default_host")]
    pub host: String,

    #[validate(nested)]
    pub database: DatabaseConfig,
}

fn default_host() -> String { "0.0.0.0".to_string() }
```

## Multi-Format Loading

```rust
impl AppConfig {
    pub fn load() -> Result<Self, ConfigError> {
        let config = Figment::new()
            .merge(Toml::file("config.toml"))
            .merge(Yaml::file("config.yaml"))
            .merge(Json::file("config.json"))
            .merge(Env::prefixed("APP_"))
            .extract()?;

        config.validate()
            .map_err(ConfigError::Validation)?;

        Ok(config)
    }
}
```

## Hot-Reloading Pattern

```rust
use arc_swap::ArcSwap;
use notify::{RecommendedWatcher, Watcher};

pub struct ConfigManager<T> {
    current: Arc<ArcSwap<T>>,
    reload_tx: broadcast::Sender<Arc<T>>,
}

impl<T> ConfigManager<T> {
    pub fn get(&self) -> Arc<T> {
        self.current.load_full()
    }

    pub fn update(&self, new_config: T) {
        let new = Arc::new(new_config);
        self.current.store(new.clone());
        let _ = self.reload_tx.send(new);
    }

    pub fn subscribe(&self) -> broadcast::Receiver<Arc<T>> {
        self.reload_tx.subscribe()
    }
}
```

## Environment-Based Config

```rust
#[derive(Debug, Clone, Copy)]
pub enum Environment {
    Development,
    Testing,
    Production,
}

impl AppConfig {
    pub fn for_environment(env: Environment) -> Result<Self, ConfigError> {
        let figment = match env {
            Environment::Development =>
                Figment::new().merge(Yaml::file("config/development.yaml")),
            Environment::Production =>
                Figment::new().merge(Yaml::file("config/production.yaml")),
            _ => Figment::new(),
        };

        figment.merge(Env::prefixed("APP_")).extract()
    }
}
```

## Anti-Patterns to Avoid

- Hardcoded configuration values
- Missing validation on config loading
- No environment variable overrides
- Single format only support
- Blocking file reads without caching

## See Also

- `reference.md` - Complete patterns, testing, and error handling
