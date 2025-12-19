---
description: >-
  Use when designing Rust APIs, function signatures, or public interfaces.
  Covers ergonomic API design, flexible input types (AsRef, Into), builder patterns
  with TypedBuilder, error handling design, and trait design principles.
  Triggers: API design, function signature, builder pattern, trait design, public API.
---

# Rust API Design

## Quick Reference

This skill provides guidance for creating ergonomic, maintainable Rust APIs.

## Key Principles

### Flexible Function Signatures
```rust
use std::path::Path;

// Accept flexible input types with AsRef
pub fn read_config<P: AsRef<Path>>(path: P) -> Result<Config, ConfigError> {
    let path = path.as_ref();
    // Implementation
}

// Use Into for string-like parameters
pub fn create_user<S: Into<String>>(name: S, email: S) -> Result<User, UserError> {
    let name = name.into();
    let email = email.into();
    // Implementation
}

// Prefer borrowing when ownership not needed
pub fn validate_email(email: &str) -> Result<(), ValidationError> {
    // Implementation
}
```

### Builder Pattern with TypedBuilder
```rust
use typed_builder::TypedBuilder;

#[derive(Debug, TypedBuilder)]
pub struct HttpClient {
    #[builder(setter(into))]
    base_url: String,

    #[builder(default = Duration::from_secs(30))]
    timeout: Duration,

    #[builder(default, setter(strip_option))]
    proxy: Option<String>,
}

// Usage
let client = HttpClient::builder()
    .base_url("https://api.example.com")
    .timeout(Duration::from_secs(60))
    .build();
```

### Error Design
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ApiError {
    #[error("Network error: {source}")]
    Network { #[from] source: reqwest::Error },

    #[error("Resource not found: {resource_type} with id {id}")]
    NotFound { resource_type: String, id: String },

    #[error("Rate limit exceeded: retry after {retry_after} seconds")]
    RateLimit { retry_after: u64 },
}

impl ApiError {
    pub fn is_retryable(&self) -> bool {
        matches!(self, ApiError::Network { .. } | ApiError::RateLimit { .. })
    }
}

// Domain-specific result type
pub type ApiResult<T> = Result<T, ApiError>;
```

### Trait Design
```rust
// Single responsibility traits
pub trait Serializable {
    fn serialize(&self) -> Result<Vec<u8>, SerializationError>;
}

pub trait Cacheable {
    type Key;
    fn cache_key(&self) -> Self::Key;
    fn cache_ttl(&self) -> Option<Duration>;
}

// Composable traits with default implementations
pub trait Timestamped {
    fn created_at(&self) -> DateTime<Utc>;
    fn updated_at(&self) -> DateTime<Utc>;

    fn age(&self) -> Duration {
        Utc::now().signed_duration_since(self.created_at())
            .to_std().unwrap_or_default()
    }
}
```

## Technology Stack

```toml
[dependencies]
typed-builder = "0.21"
thiserror = "2.0"
serde = { version = "1.0", features = ["derive"] }
```

## Anti-Patterns

- Overly generic signatures without clear benefit
- Missing error context in Result types
- Manual builder implementation (use typed-builder)
- Exposing internal implementation details
- Not providing domain-specific Result type aliases

## See Also

- `reference.md` - Complete patterns, extension traits, async API patterns
- `../core/` - Type system and code quality standards
- `../errors/` - Detailed error handling patterns
