# Rust Core Reference

## Code Quality Checklist

```markdown
### Code Quality Verification
- [ ] Uses Rust 2024 edition
- [ ] No `unsafe` code blocks
- [ ] No `unwrap()` or `expect()` in production code
- [ ] All data structures use `#[serde(rename_all = "camelCase")]`
- [ ] Files organized by functionality, not type
- [ ] Meaningful names (no "Impl" suffixes)
- [ ] Functions ≤ 150 lines
- [ ] Files ≤ 500 lines (excluding tests)
- [ ] Unit tests in same file as implementation
- [ ] `cargo build` passes
- [ ] `cargo test` passes
- [ ] `cargo clippy` passes with no warnings
- [ ] Public APIs documented with examples
```

## Dependency Management

### Workspace Dependencies Priority
```toml
# Always prefer workspace dependencies first
[dependencies]
tokio = { workspace = true }
serde = { workspace = true, features = ["derive"] }
```

### Standard Crate Recommendations

#### Core Utilities
```toml
anyhow = "1.0"                    # Simple error handling
thiserror = "2.0"                 # Structured error types
derive_more = { version = "2", features = ["full"] }
typed-builder = "0.21"            # Builder pattern
uuid = { version = "1.17", features = ["v4", "v7", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
```

#### Async/Concurrency
```toml
tokio = { version = "1.45", features = [
    "macros", "rt-multi-thread", "signal", "sync", "fs", "net", "time"
] }
async-trait = "0.1"
futures = "0.3"
dashmap = { version = "6", features = ["serde"] }
```

#### Serialization
```toml
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serde_yaml = "0.9"
```

### Feature Flag Strategy
```toml
# Only enable needed features
reqwest = { version = "0.12", default-features = false, features = [
    "rustls-tls-webpki-roots",
    "json",
    "gzip"
] }
```

### Dependency Checklist
```markdown
- [ ] Uses workspace dependencies when available
- [ ] Features flags are minimal and documented
- [ ] Prefers rustls over native-tls
- [ ] Uses latest stable versions
- [ ] No duplicate dependencies across workspace
- [ ] Dev dependencies separated from runtime deps
```

## Type System Patterns

### Validated Types with Builder
```rust
use typed_builder::TypedBuilder;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, TypedBuilder, Validate)]
#[serde(rename_all = "camelCase")]
pub struct Email {
    #[validate(email)]
    #[builder(setter(into))]
    value: String,
}

impl Email {
    pub fn new(value: impl Into<String>) -> Result<Self, ValidationError> {
        let email = Self { value: value.into() };
        email.validate()?;
        Ok(email)
    }
}
```

### Error Modeling with Enums
```rust
#[derive(thiserror::Error, Debug)]
pub enum UserServiceError {
    #[error("User not found: {user_id}")]
    NotFound { user_id: UserId },

    #[error("Email already exists: {email}")]
    EmailExists { email: Email },

    #[error("Database error: {source}")]
    Database {
        #[from]
        source: sqlx::Error,
    },

    #[error("Validation error: {message}")]
    Validation { message: String },
}

impl UserServiceError {
    pub fn is_retryable(&self) -> bool {
        matches!(self, Self::Database { .. })
    }

    pub fn error_code(&self) -> &'static str {
        match self {
            Self::NotFound { .. } => "USER_NOT_FOUND",
            Self::EmailExists { .. } => "EMAIL_EXISTS",
            Self::Database { .. } => "DATABASE_ERROR",
            Self::Validation { .. } => "VALIDATION_ERROR",
        }
    }
}
```

### State Machine with Enums
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "status", content = "data", rename_all = "camelCase")]
pub enum OrderStatus {
    Pending { items: Vec<OrderItem> },
    Processing { estimated_completion: DateTime<Utc> },
    Shipped { tracking_number: String, carrier: String },
    Delivered { delivery_time: DateTime<Utc> },
    Cancelled { reason: String, refund_issued: bool },
}

impl OrderStatus {
    pub fn can_cancel(&self) -> bool {
        matches!(self, Self::Pending { .. } | Self::Processing { .. })
    }
}
```

### Type System Checklist
```markdown
- [ ] Uses newtype pattern for domain concepts
- [ ] Phantom types for compile-time state tracking
- [ ] Associated types vs generics chosen appropriately
- [ ] Enums model state machines correctly
- [ ] Option/Result combinators used over unwrap
- [ ] Builder patterns enforce required fields
- [ ] No primitive obsession (avoid String/i32 for everything)
```

## Performance Optimization

### Memory Management
```rust
use std::borrow::Cow;

// Use Cow for flexible string handling
pub fn process_text<'a>(input: &'a str) -> Cow<'a, str> {
    if input.contains("old") {
        Cow::Owned(input.replace("old", "new"))
    } else {
        Cow::Borrowed(input)
    }
}

// Pre-allocate with known capacity
pub fn build_large_string(items: &[&str]) -> String {
    let total_len = items.iter().map(|s| s.len()).sum::<usize>();
    let mut result = String::with_capacity(total_len + items.len() - 1);
    // ...
    result
}
```

### Iterator Optimization
```rust
// Chain iterators for efficiency
pub fn process_and_filter(data: &[i32]) -> Vec<i32> {
    data.iter()
        .filter(|&&x| x > 0)
        .map(|&x| x * 2)
        .filter(|&x| x < 1000)
        .collect()
}

// Avoid collecting intermediate results
```

### Cargo.toml Optimizations
```toml
[profile.release]
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

### Performance Checklist
```markdown
- [ ] Pre-allocate collections with known capacity
- [ ] Use appropriate data structures (HashMap vs Vec for lookups)
- [ ] Leverage iterator chains instead of intermediate collections
- [ ] Use Cow for flexible string handling
- [ ] Enable LTO and appropriate optimization flags
- [ ] Avoid unnecessary clones and allocations
```

## Security Patterns

### Input Validation
```rust
use validator::{Validate, ValidationError};

#[derive(Debug, Clone, Validate)]
pub struct UserRegistration {
    #[validate(email, message = "Invalid email format")]
    pub email: String,

    #[validate(length(min = 8, max = 128))]
    pub password: String,

    #[validate(length(min = 2, max = 50))]
    pub username: String,
}
```

### Password Hashing (Argon2)
```rust
use argon2::{Argon2, PasswordHash, PasswordHasher, PasswordVerifier};
use argon2::password_hash::{rand_core::OsRng, SaltString};

pub fn hash_password(password: &str) -> Result<String, SecurityError> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    let password_hash = argon2
        .hash_password(password.as_bytes(), &salt)
        .map_err(|_| SecurityError::HashingError)?;
    Ok(password_hash.to_string())
}
```

### Secrets Management
```rust
use zeroize::Zeroize;

#[derive(Zeroize)]
#[zeroize(drop)]
pub struct Secret {
    value: String,
}

impl Secret {
    pub fn from_env(key: &str) -> Result<Self, SecurityError> {
        let value = std::env::var(key)
            .map_err(|_| SecurityError::MissingSecret)?;
        Ok(Self { value })
    }
    // Never implement Display or Debug for secrets
}
```

### Security Checklist
```markdown
- [ ] All user inputs are validated and sanitized
- [ ] SQL queries use parameterized statements
- [ ] Passwords are hashed with Argon2
- [ ] Secrets are loaded from environment variables
- [ ] Sensitive data structures implement Zeroize
- [ ] Error messages don't leak sensitive information
```

## API Design Patterns

### Ergonomic Function Signatures
```rust
// Accept flexible input types
pub fn read_config<P: AsRef<Path>>(path: P) -> Result<Config, ConfigError> {
    let path = path.as_ref();
    // ...
}

// Use Into for string-like parameters
pub fn create_user<S: Into<String>>(name: S, email: S) -> Result<User, UserError> {
    let name = name.into();
    let email = email.into();
    // ...
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
```

### API Design Checklist
```markdown
- [ ] Function signatures accept flexible input types (AsRef, Into)
- [ ] Error types are well-structured with proper context
- [ ] Builder pattern used for complex configuration
- [ ] Public API is well-documented with examples
- [ ] Dependencies are injected for testability
```

## Design Patterns

### Strategy Pattern with Enums
```rust
#[derive(Debug, Clone)]
pub enum AuthStrategy {
    Bearer { token: String },
    ApiKey { key: String, header: String },
    Basic { username: String, password: String },
}

impl AuthStrategy {
    pub fn apply_to_request(&self, request: &mut Request) -> Result<(), AuthError> {
        match self {
            AuthStrategy::Bearer { token } => { /* ... */ }
            AuthStrategy::ApiKey { key, header } => { /* ... */ }
            AuthStrategy::Basic { username, password } => { /* ... */ }
        }
        Ok(())
    }
}
```

### Factory Pattern with Associated Types
```rust
pub trait ConnectionFactory {
    type Connection;
    type Config;
    type Error;

    fn create_connection(config: Self::Config) -> Result<Self::Connection, Self::Error>;
}
```

### Observer Pattern with Async
```rust
use tokio::sync::broadcast;

#[derive(Debug, Clone)]
pub enum DomainEvent {
    UserCreated { user_id: UserId, email: String },
    UserUpdated { user_id: UserId, changes: Vec<String> },
    OrderPlaced { order_id: OrderId, user_id: UserId },
}

pub struct EventBus {
    sender: broadcast::Sender<DomainEvent>,
}

impl EventBus {
    pub async fn publish(&self, event: DomainEvent) -> Result<(), EventError> {
        let _ = self.sender.send(event);
        Ok(())
    }
}
```

## Testing Standards

### Unit Test Placement
```rust
// Always place unit tests in the same file
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_user_validation() {
        let validator = UserValidator::new();
        let user = User::default();
        assert!(validator.validate(&user).is_ok());
    }
}
```

### Test Naming
```rust
#[test]
fn test_valid_email_passes_validation() { /* ... */ }

#[test]
fn test_empty_email_returns_error() { /* ... */ }
```

## Module Organization

### Feature-based Modules
```
src/
├── user/
│   ├── mod.rs
│   ├── service.rs
│   ├── repository.rs
│   └── validator.rs
├── product/
│   ├── mod.rs
│   ├── catalog.rs
│   └── pricing.rs
└── auth/
    ├── mod.rs
    ├── token.rs
    └── session.rs
```

---
*Source: cursor-rust-rules core/*.mdc files*
