---
description: >-
  Use when implementing common utilities in Rust. Covers JWT authentication,
  random generation, typed-builder patterns, derive_more macros, and CLI tools.
  Triggers: JWT, authentication, random, derive_more, typed-builder, utility.
---

# Rust Utilities

## Quick Reference

Common utility patterns for Rust applications including authentication, random generation, and enhanced derives.

## JWT Authentication

```rust
use jsonwebtoken::{encode, decode, Header, Algorithm, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,
    pub exp: usize,
    pub iat: usize,
    pub roles: Vec<String>,
}

pub struct JwtService {
    encoding_key: EncodingKey,
    decoding_key: DecodingKey,
    validation: Validation,
}

impl JwtService {
    pub fn new(secret: &[u8]) -> Self {
        Self {
            encoding_key: EncodingKey::from_secret(secret),
            decoding_key: DecodingKey::from_secret(secret),
            validation: Validation::new(Algorithm::HS256),
        }
    }

    pub fn generate_token(&self, user_id: &str, roles: Vec<String>) -> Result<String, JwtError> {
        let now = chrono::Utc::now().timestamp() as usize;
        let claims = Claims {
            sub: user_id.to_string(),
            exp: now + 3600, // 1 hour
            iat: now,
            roles,
        };
        encode(&Header::default(), &claims, &self.encoding_key)
            .map_err(JwtError::from)
    }

    pub fn verify_token(&self, token: &str) -> Result<Claims, JwtError> {
        decode::<Claims>(token, &self.decoding_key, &self.validation)
            .map(|data| data.claims)
            .map_err(JwtError::from)
    }
}
```

## Random Generation

```rust
use rand::{Rng, thread_rng, distributions::Alphanumeric};

pub struct RandomGenerator;

impl RandomGenerator {
    pub fn alphanumeric(length: usize) -> String {
        thread_rng()
            .sample_iter(&Alphanumeric)
            .take(length)
            .map(char::from)
            .collect()
    }

    pub fn uuid_v4() -> String {
        uuid::Uuid::new_v4().to_string()
    }

    pub fn uuid_v7() -> String {
        uuid::Uuid::now_v7().to_string()
    }

    pub fn range(min: i32, max: i32) -> i32 {
        thread_rng().gen_range(min..=max)
    }
}
```

## TypedBuilder Pattern

```rust
use typed_builder::TypedBuilder;

#[derive(Debug, TypedBuilder)]
pub struct UserConfig {
    #[builder(setter(into))]
    username: String,

    #[builder(setter(into))]
    email: String,

    #[builder(default = 18)]
    age: u8,

    #[builder(default, setter(strip_option))]
    phone: Option<String>,

    #[builder(default = vec!["user".to_string()])]
    roles: Vec<String>,
}

// Usage
let config = UserConfig::builder()
    .username("john")
    .email("john@example.com")
    .age(25)
    .phone("123-456-7890")
    .build();
```

## derive_more Macros

```rust
use derive_more::{Display, From, Into, Deref, DerefMut, Constructor};

#[derive(Debug, Clone, Display, From, Into, Deref, DerefMut)]
#[display("{}", _0)]
pub struct Email(String);

#[derive(Debug, Clone, Display, From, Constructor)]
#[display("UserId({})", id)]
pub struct UserId {
    id: uuid::Uuid,
}

#[derive(Debug, Display)]
pub enum ApiError {
    #[display("Not found: {}", resource)]
    NotFound { resource: String },

    #[display("Validation failed: {}", message)]
    Validation { message: String },
}
```

## Technology Stack

```toml
[dependencies]
jsonwebtoken = "9.3"
uuid = { version = "1.17", features = ["v4", "v7", "serde"] }
rand = "0.8"
typed-builder = "0.21"
derive_more = { version = "2", features = ["full"] }
chrono = { version = "0.4", features = ["serde"] }
```

## See Also

- `reference.md` - Complete patterns and CLI tools
- `../api-design/` - API design with builders
- `../core/` - Type system patterns
