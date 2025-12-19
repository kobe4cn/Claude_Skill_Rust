---
description: >-
  Use when setting up logging, configuration, or templating in Rust. Covers tracing
  ecosystem (not env_logger), YAML configuration with figment, MiniJinja templating
  (not Handlebars), and JSONPath data extraction.
  Triggers: tracing, logging, configuration, YAML, template, MiniJinja, figment.
---

# Rust Tools & Configuration

## Quick Reference

Essential tools and configuration patterns for modern Rust applications.

## Tracing Ecosystem (Not env_logger)

```rust
use tracing::{info, error, span, Level};
use tracing_subscriber::{
    fmt, layer::SubscriberExt, util::SubscriberInitExt, EnvFilter,
};

pub fn init_logging() {
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("info")))
        .with(fmt::layer()
            .with_target(true)
            .with_thread_ids(true)
            .with_level(true))
        .init();
}

// Usage with structured fields
#[tracing::instrument(skip(service), fields(user_id = %user_id))]
pub async fn process_request(user_id: &str, service: &Service) -> Result<Response, Error> {
    info!("Processing request");

    let result = service.execute().await.map_err(|e| {
        error!(error = %e, "Request failed");
        e
    })?;

    info!(result_id = %result.id, "Request completed");
    Ok(result)
}
```

## YAML Configuration with Figment

```rust
use figment::{Figment, providers::{Env, Format, Yaml}};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct AppConfig {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub logging: LogConfig,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
}

impl AppConfig {
    pub fn load() -> Result<Self, figment::Error> {
        let env = std::env::var("ENVIRONMENT").unwrap_or_else(|_| "development".to_string());

        Figment::new()
            .merge(Yaml::file(format!("config/{}.yaml", env)))
            .merge(Env::prefixed("APP_").split("_"))
            .extract()
    }
}
```

## MiniJinja Templating (Not Handlebars)

```rust
use minijinja::{Environment, context};

pub struct TemplateEngine {
    env: Environment<'static>,
}

impl TemplateEngine {
    pub fn new() -> Result<Self, minijinja::Error> {
        let mut env = Environment::new();
        env.set_loader(minijinja::path_loader("templates"));

        // Custom filters
        env.add_filter("currency", |value: f64| format!("${:.2}", value));

        Ok(Self { env })
    }

    pub fn render_email(&self, template: &str, user: &User) -> Result<String, minijinja::Error> {
        let tmpl = self.env.get_template(template)?;
        tmpl.render(context! {
            user => user,
            app_name => "MyApp",
        })
    }
}
```

## Technology Stack

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
figment = { version = "0.10", features = ["yaml", "env"] }
minijinja = { version = "2", features = ["loader"] }
serde = { version = "1.0", features = ["derive"] }
serde_yaml = "0.9"
```

## Anti-Patterns

- Using `env_logger` instead of `tracing`
- Using TOML for complex nested configuration
- Using Handlebars instead of MiniJinja
- Hardcoding secrets in config files
- Using `println!` for logging in production

## See Also

- `reference.md` - Complete patterns, file rotation, JSONPath
- `../observability/` - Metrics and distributed tracing
- `../config/` - Hot-reload configuration patterns
