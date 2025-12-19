# Rust Tools & Configuration Reference

## Structured Logging with Tracing

```rust
use tracing::{info, error, warn, debug, span, Level};
use tracing_subscriber::{
    fmt::{self, time::ChronoUtc},
    layer::SubscriberExt,
    util::SubscriberInitExt,
    EnvFilter, Registry,
};
use tracing_appender::{non_blocking, rolling};

pub struct LogConfig {
    pub level: String,
    pub log_dir: String,
    pub json_format: bool,
}

pub fn init_logging(config: &LogConfig) -> Result<(), Box<dyn std::error::Error>> {
    // File appender with daily rotation
    let file_appender = rolling::daily(&config.log_dir, "app.log");
    let (file_writer, _guard) = non_blocking(file_appender);

    // Console layer
    let console_layer = fmt::layer()
        .with_target(true)
        .with_timer(ChronoUtc::rfc_3339())
        .with_level(true)
        .with_thread_ids(true);

    // File layer (JSON format)
    let file_layer = fmt::layer()
        .json()
        .with_timer(ChronoUtc::rfc_3339())
        .with_writer(file_writer);

    // Environment filter
    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new(&config.level));

    Registry::default()
        .with(filter)
        .with(console_layer)
        .with(file_layer)
        .init();

    Ok(())
}

// Instrumented function with structured fields
#[tracing::instrument(skip(service), fields(user_id = %user_id))]
pub async fn process_user_registration(
    user_id: &str,
    service: &UserService,
) -> Result<User, ServiceError> {
    let span = span!(Level::INFO, "user_registration", user_id = %user_id);
    let _enter = span.enter();

    info!("Starting user registration");

    let user = service.create_user(user_id).await.map_err(|e| {
        error!(error = %e, "Failed to create user");
        e
    })?;

    info!(
        user_id = %user.id,
        email = %user.email,
        "User registration completed"
    );

    Ok(user)
}
```

## YAML Configuration with Figment

```yaml
# config/development.yaml
server:
  host: "127.0.0.1"
  port: 8080
  workers: 1

database:
  url: "postgresql://user:pass@localhost/app_dev"
  maxConnections: 10
  minConnections: 2
  connectTimeout: 30s
  idleTimeout: 600s

logging:
  level: "debug"
  format: "pretty"
  logDir: "./logs"

features:
  enableRegistration: true
  enablePasswordReset: true
  maintenanceMode: false

security:
  jwtSecret: "${JWT_SECRET}"  # From environment
  sessionTimeout: 3600
```

```rust
use figment::{Figment, providers::{Env, Format, Yaml}};
use serde::{Deserialize, Serialize};
use std::time::Duration;

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct AppConfig {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub logging: LogConfig,
    pub features: FeatureFlags,
    pub security: SecurityConfig,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct DatabaseConfig {
    pub url: String,
    pub max_connections: u32,
    pub min_connections: u32,
    #[serde(with = "humantime_serde")]
    pub connect_timeout: Duration,
    #[serde(with = "humantime_serde")]
    pub idle_timeout: Duration,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct FeatureFlags {
    pub enable_registration: bool,
    pub enable_password_reset: bool,
    pub maintenance_mode: bool,
}

impl AppConfig {
    pub fn load() -> Result<Self, figment::Error> {
        let env = std::env::var("ENVIRONMENT").unwrap_or_else(|_| "development".to_string());

        Figment::new()
            .merge(Yaml::file(format!("config/{}.yaml", env)))
            .merge(Env::prefixed("APP_").split("_"))
            .extract()
    }

    pub fn validate(&self) -> Result<(), ConfigError> {
        if self.server.port == 0 {
            return Err(ConfigError::InvalidValue("Server port cannot be 0".into()));
        }
        if self.database.max_connections < self.database.min_connections {
            return Err(ConfigError::InvalidValue(
                "max_connections must be >= min_connections".into()
            ));
        }
        Ok(())
    }
}
```

## MiniJinja Templating

```rust
use minijinja::{Environment, Error as TemplateError, Value};

pub struct TemplateEngine {
    env: Environment<'static>,
}

impl TemplateEngine {
    pub fn new() -> Result<Self, TemplateError> {
        let mut env = Environment::new();

        // Load templates from directory
        env.set_loader(minijinja::path_loader("templates"));

        // Custom filters
        env.add_filter("currency", currency_filter);
        env.add_filter("date_format", date_format_filter);
        env.add_filter("truncate", truncate_filter);

        // Custom functions
        env.add_function("asset_url", asset_url_function);
        env.add_function("config", config_function);

        Ok(Self { env })
    }

    pub fn render_email(&self, template: &str, ctx: &EmailContext) -> Result<String, TemplateError> {
        let tmpl = self.env.get_template(template)?;
        tmpl.render(ctx)
    }
}

fn currency_filter(value: f64) -> String {
    format!("${:.2}", value)
}

fn date_format_filter(value: &str, format: Option<&str>) -> Result<String, TemplateError> {
    let fmt = format.unwrap_or("%Y-%m-%d");
    // Parse and format date
    Ok(value.to_string()) // Simplified
}

fn truncate_filter(value: String, length: Option<i64>) -> String {
    let len = length.unwrap_or(100) as usize;
    if value.len() <= len {
        value
    } else {
        format!("{}...", &value[..len.saturating_sub(3)])
    }
}

fn asset_url_function(path: &str) -> String {
    format!("/assets/{}", path)
}

fn config_function(key: &str) -> Value {
    match key {
        "app.name" => Value::from("My Application"),
        "app.version" => Value::from("1.0.0"),
        _ => Value::UNDEFINED,
    }
}
```

### Template Example

```jinja2
{# templates/emails/welcome.html #}
{% extends "emails/base.html" %}

{% block content %}
<h1>Welcome, {{ user.firstName }}!</h1>

<p>Thank you for registering with {{ config('app.name') }}.</p>

<div class="order-summary">
    <h2>Your Order #{{ order.orderNumber }}</h2>
    <p><strong>Total:</strong> {{ order.totalAmount | currency }}</p>
    <p><strong>Date:</strong> {{ order.createdAt | date_format("%B %d, %Y") }}</p>

    <ul>
    {% for item in order.items %}
        <li>{{ item.productName }} - Qty: {{ item.quantity }}</li>
    {% endfor %}
    </ul>
</div>

<a href="{{ verificationUrl }}" class="btn">Verify Email</a>
{% endblock %}
```

## JSONPath Data Extraction

```rust
use jsonpath_rust::JsonPathQuery;
use serde_json::{Value, json};

pub struct DataTransformer;

impl DataTransformer {
    pub fn extract_user_emails(api_response: &Value) -> Result<Vec<String>, TransformError> {
        let emails = api_response
            .path("$.data.users[*].contact.email")?
            .as_array()
            .ok_or(TransformError::InvalidPath)?
            .iter()
            .filter_map(|v| v.as_str())
            .map(String::from)
            .collect();

        Ok(emails)
    }

    pub fn extract_order_summary(order_data: &Value) -> Result<OrderSummary, TransformError> {
        let order_id = order_data.path("$.order.id")?
            .as_str()
            .ok_or(TransformError::MissingField("order.id"))?
            .to_string();

        let total = order_data.path("$.payment.total")?
            .as_f64()
            .ok_or(TransformError::MissingField("payment.total"))?;

        let items: Vec<String> = order_data
            .path("$.items[*].product.name")?
            .as_array()
            .unwrap_or(&vec![])
            .iter()
            .filter_map(|v| v.as_str())
            .map(String::from)
            .collect();

        Ok(OrderSummary { order_id, total, items })
    }
}
```

## Tools & Config Checklist

```markdown
### Tools & Configuration Verification
- [ ] Uses tracing ecosystem (not env_logger)
- [ ] YAML configuration files (not TOML for complex configs)
- [ ] Environment variable overrides for sensitive data
- [ ] Configuration validation on startup
- [ ] MiniJinja templating (not Handlebars)
- [ ] Custom filters and functions for templates
- [ ] Structured logging with spans and events
- [ ] File rotation for production logs
- [ ] JSON format for machine-readable logs
- [ ] Template inheritance for reusability
```

---
*Source: .cursor/rules/rust/features/tools-and-config.mdc*
