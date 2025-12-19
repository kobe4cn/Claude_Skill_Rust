# Rust API Design Reference

## Ergonomic Function Signatures

### Flexible Input Types
```rust
use std::path::Path;

// AsRef for path-like parameters
pub fn read_config<P: AsRef<Path>>(path: P) -> Result<Config, ConfigError> {
    let path = path.as_ref();
    let content = std::fs::read_to_string(path)?;
    // Parse and return
}

// Into for string-like parameters
pub fn create_user<S: Into<String>>(name: S, email: S) -> Result<User, UserError> {
    let name = name.into();
    let email = email.into();
    // Validation and creation
}

// Borrowing when ownership not needed
pub fn validate_email(email: &str) -> Result<(), ValidationError> {
    if !email.contains('@') {
        return Err(ValidationError::InvalidEmail);
    }
    Ok(())
}

// Return owned data when caller needs ownership
pub fn generate_token() -> String {
    uuid::Uuid::new_v4().to_string()
}
```

### Extension Traits
```rust
// Extension traits for external types
pub trait StringExtensions {
    fn is_valid_email(&self) -> bool;
    fn to_snake_case(&self) -> String;
    fn truncate_with_ellipsis(&self, max_len: usize) -> String;
}

impl StringExtensions for str {
    fn is_valid_email(&self) -> bool {
        self.contains('@') && self.contains('.')
    }

    fn to_snake_case(&self) -> String {
        self.chars()
            .map(|c| if c.is_uppercase() {
                format!("_{}", c.to_lowercase())
            } else {
                c.to_string()
            })
            .collect::<String>()
            .trim_start_matches('_')
            .to_string()
    }

    fn truncate_with_ellipsis(&self, max_len: usize) -> String {
        if self.len() <= max_len {
            self.to_string()
        } else {
            format!("{}...", &self[..max_len.saturating_sub(3)])
        }
    }
}

// Extension traits for Result types
pub trait ResultExtensions<T, E> {
    fn log_error(self) -> Self;
}

impl<T, E: std::fmt::Debug> ResultExtensions<T, E> for Result<T, E> {
    fn log_error(self) -> Self {
        if let Err(ref e) = self {
            tracing::error!("Operation failed: {:?}", e);
        }
        self
    }
}
```

## Builder Pattern Implementation

### TypedBuilder for Complex Configuration
```rust
use typed_builder::TypedBuilder;
use std::time::Duration;
use std::collections::HashMap;

#[derive(Debug, TypedBuilder)]
pub struct HttpClient {
    #[builder(setter(into))]
    base_url: String,

    #[builder(default = Duration::from_secs(30))]
    timeout: Duration,

    #[builder(default)]
    headers: HashMap<String, String>,

    #[builder(default, setter(strip_option))]
    proxy: Option<String>,

    #[builder(default = false)]
    verify_ssl: bool,
}

impl HttpClient {
    // Simple constructor for common cases
    pub fn new<S: Into<String>>(base_url: S) -> Self {
        Self::builder()
            .base_url(base_url)
            .build()
    }

    // Factory method with auth
    pub fn with_auth<S: Into<String>>(base_url: S, token: S) -> Self {
        let mut headers = HashMap::new();
        headers.insert(
            "Authorization".to_string(),
            format!("Bearer {}", token.into())
        );

        Self::builder()
            .base_url(base_url)
            .headers(headers)
            .build()
    }
}
```

## Error Handling Design

### Well-Structured Error Hierarchy
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ApiError {
    #[error("Network error: {source}")]
    Network {
        #[from]
        source: reqwest::Error,
    },

    #[error("Invalid request: {message}")]
    InvalidRequest { message: String },

    #[error("Authentication failed")]
    Authentication,

    #[error("Resource not found: {resource_type} with id {id}")]
    NotFound { resource_type: String, id: String },

    #[error("Rate limit exceeded: retry after {retry_after} seconds")]
    RateLimit { retry_after: u64 },

    #[error("Server error: {status_code}")]
    Server { status_code: u16 },
}

impl ApiError {
    pub fn is_retryable(&self) -> bool {
        matches!(
            self,
            ApiError::Network { .. }
                | ApiError::RateLimit { .. }
                | ApiError::Server { status_code } if *status_code >= 500
        )
    }

    pub fn retry_after(&self) -> Option<Duration> {
        match self {
            ApiError::RateLimit { retry_after } => {
                Some(Duration::from_secs(*retry_after))
            }
            _ => None,
        }
    }

    pub fn error_code(&self) -> &'static str {
        match self {
            ApiError::Network { .. } => "NETWORK_ERROR",
            ApiError::InvalidRequest { .. } => "INVALID_REQUEST",
            ApiError::Authentication => "AUTH_ERROR",
            ApiError::NotFound { .. } => "NOT_FOUND",
            ApiError::RateLimit { .. } => "RATE_LIMITED",
            ApiError::Server { .. } => "SERVER_ERROR",
        }
    }
}

// Domain-specific result type
pub type ApiResult<T> = Result<T, ApiError>;
```

## Trait Design Patterns

### Cohesive Traits
```rust
// Single responsibility traits
pub trait Serializable {
    fn serialize(&self) -> Result<Vec<u8>, SerializationError>;
    fn deserialize(data: &[u8]) -> Result<Self, SerializationError>
    where
        Self: Sized;
}

pub trait Cacheable {
    type Key;
    fn cache_key(&self) -> Self::Key;
    fn cache_ttl(&self) -> Option<Duration>;
}

// Composable repository traits
pub trait Repository<T> {
    type Error;
    type Id;

    async fn find_by_id(&self, id: Self::Id) -> Result<Option<T>, Self::Error>;
    async fn save(&self, entity: &T) -> Result<T, Self::Error>;
    async fn delete(&self, id: Self::Id) -> Result<bool, Self::Error>;
}

pub trait Queryable<T>: Repository<T> {
    type Query;
    type Page;

    async fn find_by_query(&self, query: Self::Query) -> Result<Vec<T>, Self::Error>;
    async fn find_paginated(
        &self,
        query: Self::Query,
        page: Self::Page
    ) -> Result<(Vec<T>, bool), Self::Error>;
}
```

## Module Organization

### Public API Structure
```rust
// lib.rs - Main library entry point
//! # MyLibrary
//!
//! A comprehensive library for handling X, Y, and Z.
//!
//! ## Quick Start
//!
//! ```rust
//! use my_library::Client;
//!
//! let client = Client::new("api-key");
//! let result = client.fetch_data().await?;
//! ```

// Re-export main public API
pub use client::Client;
pub use config::Config;
pub use error::{Error, Result};

// Re-export important types
pub use types::{User, Product, Order};

// Module declarations
mod client;
mod config;
mod error;
mod types;

// Internal modules (not re-exported)
mod internal {
    pub mod auth;
    pub mod http;
}

// Prelude for convenient imports
pub mod prelude {
    pub use crate::{Client, Config, Error, Result};
    pub use crate::types::*;
}

// Feature-gated modules
#[cfg(feature = "async")]
pub mod async_client;
```

## Documentation Standards

```rust
/// A client for interacting with the Example API.
///
/// The `Client` provides methods for authentication, data retrieval,
/// and resource management.
///
/// # Examples
///
/// Basic usage:
///
/// ```rust
/// use my_library::Client;
///
/// # tokio_test::block_on(async {
/// let client = Client::new("your-api-key");
/// let users = client.list_users().await?;
/// # Ok::<(), Box<dyn std::error::Error>>(())
/// # });
/// ```
///
/// # Errors
///
/// Returns an error if:
/// * The API key is invalid (`Error::Authentication`)
/// * The request times out (`Error::Network`)
pub struct Client {
    api_key: String,
    config: Config,
}
```

## Async API Patterns

### Async Iterator and Stream Design
```rust
use futures::Stream;
use std::pin::Pin;

// Paginated stream for async iteration
pub struct PaginatedStream<T> {
    client: Arc<Client>,
    query: Query,
    current_page: Option<String>,
    buffer: VecDeque<T>,
    exhausted: bool,
}

impl<T> PaginatedStream<T> {
    pub fn new(client: Arc<Client>, query: Query) -> Self {
        Self {
            client,
            query,
            current_page: None,
            buffer: VecDeque::new(),
            exhausted: false,
        }
    }
}
```

## Testable API Structure

### Dependency Injection for Testing
```rust
// Trait for HTTP operations
pub trait HttpClientTrait: Send + Sync {
    async fn get(&self, url: &str) -> Result<Response, HttpError>;
    async fn post(&self, url: &str, body: Vec<u8>) -> Result<Response, HttpError>;
}

// Generic client using trait
pub struct Client<H: HttpClientTrait> {
    http_client: H,
    config: Config,
}

impl<H: HttpClientTrait> Client<H> {
    pub fn new(http_client: H, config: Config) -> Self {
        Self { http_client, config }
    }

    pub async fn fetch_user(&self, id: &str) -> Result<User, ApiError> {
        let url = format!("{}/users/{}", self.config.base_url, id);
        let response = self.http_client.get(&url).await?;
        // Parse response
    }
}

// Mock for testing
#[cfg(test)]
pub struct MockHttpClient {
    responses: HashMap<String, Result<Response, HttpError>>,
}

#[cfg(test)]
impl HttpClientTrait for MockHttpClient {
    async fn get(&self, url: &str) -> Result<Response, HttpError> {
        self.responses
            .get(&format!("GET {}", url))
            .cloned()
            .unwrap_or(Err(HttpError::NotFound))
    }
    // ...
}
```

## API Design Checklist

```markdown
### API Design Verification
- [ ] Function signatures accept flexible input types (AsRef, Into)
- [ ] Error types are well-structured with proper context
- [ ] Builder pattern used for complex configuration
- [ ] Traits have single responsibility and clear contracts
- [ ] Public API is well-documented with examples
- [ ] Dependencies are injected for testability
- [ ] Extension traits enhance existing types ergonomically
- [ ] Module organization follows convention
- [ ] Feature gates are used appropriately
- [ ] Error handling provides actionable information
- [ ] API follows Rust naming conventions
- [ ] Generic parameters have appropriate bounds
- [ ] Public API surface is minimal but complete
```

---
*Source: .cursor/rules/rust/core/api-design.mdc*
