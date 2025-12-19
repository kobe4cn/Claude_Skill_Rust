# Rust Utilities Reference

## JWT Authentication (Complete)

```rust
use jsonwebtoken::{encode, decode, Header, Algorithm, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};
use std::time::{Duration, SystemTime, UNIX_EPOCH};

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,           // Subject (user ID)
    pub exp: usize,            // Expiration time
    pub iat: usize,            // Issued at
    pub nbf: usize,            // Not before
    pub roles: Vec<String>,    // User roles
    pub token_type: TokenType, // Access or Refresh
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum TokenType {
    Access,
    Refresh,
}

pub struct TokenPair {
    pub access_token: String,
    pub refresh_token: String,
    pub expires_in: u64,
}

pub struct JwtService {
    encoding_key: EncodingKey,
    decoding_key: DecodingKey,
    access_token_ttl: Duration,
    refresh_token_ttl: Duration,
}

impl JwtService {
    pub fn new(secret: &[u8]) -> Self {
        Self {
            encoding_key: EncodingKey::from_secret(secret),
            decoding_key: DecodingKey::from_secret(secret),
            access_token_ttl: Duration::from_secs(3600),      // 1 hour
            refresh_token_ttl: Duration::from_secs(604800),   // 7 days
        }
    }

    pub fn generate_token_pair(&self, user_id: &str, roles: Vec<String>) -> Result<TokenPair, JwtError> {
        let access_token = self.create_token(user_id, roles.clone(), TokenType::Access)?;
        let refresh_token = self.create_token(user_id, roles, TokenType::Refresh)?;

        Ok(TokenPair {
            access_token,
            refresh_token,
            expires_in: self.access_token_ttl.as_secs(),
        })
    }

    fn create_token(&self, user_id: &str, roles: Vec<String>, token_type: TokenType) -> Result<String, JwtError> {
        let now = SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs() as usize;
        let ttl = match token_type {
            TokenType::Access => self.access_token_ttl,
            TokenType::Refresh => self.refresh_token_ttl,
        };

        let claims = Claims {
            sub: user_id.to_string(),
            exp: now + ttl.as_secs() as usize,
            iat: now,
            nbf: now,
            roles,
            token_type,
        };

        encode(&Header::default(), &claims, &self.encoding_key)
            .map_err(JwtError::from)
    }

    pub fn verify_access_token(&self, token: &str) -> Result<Claims, JwtError> {
        let claims = self.verify_token(token)?;
        if claims.token_type != TokenType::Access {
            return Err(JwtError::InvalidTokenType);
        }
        Ok(claims)
    }

    pub fn verify_refresh_token(&self, token: &str) -> Result<Claims, JwtError> {
        let claims = self.verify_token(token)?;
        if claims.token_type != TokenType::Refresh {
            return Err(JwtError::InvalidTokenType);
        }
        Ok(claims)
    }

    fn verify_token(&self, token: &str) -> Result<Claims, JwtError> {
        decode::<Claims>(token, &self.decoding_key, &Validation::new(Algorithm::HS256))
            .map(|data| data.claims)
            .map_err(JwtError::from)
    }

    pub fn refresh_tokens(&self, refresh_token: &str) -> Result<TokenPair, JwtError> {
        let claims = self.verify_refresh_token(refresh_token)?;
        self.generate_token_pair(&claims.sub, claims.roles)
    }
}

#[derive(thiserror::Error, Debug)]
pub enum JwtError {
    #[error("Token encoding failed: {0}")]
    Encoding(#[from] jsonwebtoken::errors::Error),

    #[error("Invalid token type")]
    InvalidTokenType,

    #[error("System time error: {0}")]
    SystemTime(#[from] std::time::SystemTimeError),
}
```

## Random Generation Utilities

```rust
use rand::{Rng, SeedableRng, thread_rng, distributions::Alphanumeric};
use rand::rngs::StdRng;

pub struct RandomGenerator;

impl RandomGenerator {
    /// Generate alphanumeric string of given length
    pub fn alphanumeric(length: usize) -> String {
        thread_rng()
            .sample_iter(&Alphanumeric)
            .take(length)
            .map(char::from)
            .collect()
    }

    /// Generate UUID v4 (random)
    pub fn uuid_v4() -> String {
        uuid::Uuid::new_v4().to_string()
    }

    /// Generate UUID v7 (time-ordered)
    pub fn uuid_v7() -> String {
        uuid::Uuid::now_v7().to_string()
    }

    /// Generate random integer in range [min, max]
    pub fn range(min: i32, max: i32) -> i32 {
        thread_rng().gen_range(min..=max)
    }

    /// Generate secure random bytes
    pub fn bytes(length: usize) -> Vec<u8> {
        let mut bytes = vec![0u8; length];
        thread_rng().fill(&mut bytes[..]);
        bytes
    }

    /// Generate deterministic random with seed (for testing)
    pub fn seeded(seed: u64) -> StdRng {
        StdRng::seed_from_u64(seed)
    }

    /// Generate random hex string
    pub fn hex(length: usize) -> String {
        Self::bytes(length / 2 + 1)
            .iter()
            .map(|b| format!("{:02x}", b))
            .collect::<String>()
            [..length]
            .to_string()
    }

    /// Generate OTP code
    pub fn otp(digits: usize) -> String {
        let max = 10_u32.pow(digits as u32);
        format!("{:0width$}", thread_rng().gen_range(0..max), width = digits)
    }
}
```

## CLI with Clap 4.0+

```rust
use clap::{Parser, Subcommand, Args, ValueEnum};

#[derive(Parser)]
#[command(name = "myapp")]
#[command(author, version, about, long_about = None)]
pub struct Cli {
    /// Enable verbose output
    #[arg(short, long, global = true)]
    verbose: bool,

    /// Configuration file path
    #[arg(short, long, default_value = "config.yaml")]
    config: String,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
pub enum Commands {
    /// Start the server
    Start(StartArgs),

    /// Manage users
    User {
        #[command(subcommand)]
        action: UserCommands,
    },

    /// Database operations
    Db {
        #[command(subcommand)]
        action: DbCommands,
    },
}

#[derive(Args)]
pub struct StartArgs {
    /// Port to listen on
    #[arg(short, long, default_value = "8080")]
    port: u16,

    /// Host to bind to
    #[arg(short = 'H', long, default_value = "127.0.0.1")]
    host: String,

    /// Environment mode
    #[arg(short, long, value_enum, default_value = "development")]
    environment: Environment,
}

#[derive(Clone, ValueEnum)]
pub enum Environment {
    Development,
    Staging,
    Production,
}

#[derive(Subcommand)]
pub enum UserCommands {
    /// List all users
    List {
        #[arg(short, long)]
        limit: Option<usize>,
    },
    /// Create a new user
    Create {
        #[arg(short, long)]
        email: String,
        #[arg(short, long)]
        name: String,
    },
    /// Delete a user
    Delete {
        #[arg(short, long)]
        id: String,
    },
}

#[derive(Subcommand)]
pub enum DbCommands {
    /// Run migrations
    Migrate,
    /// Seed database
    Seed,
    /// Reset database
    Reset {
        #[arg(long)]
        force: bool,
    },
}

// Main function
fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();

    match cli.command {
        Commands::Start(args) => {
            println!("Starting server on {}:{}", args.host, args.port);
        }
        Commands::User { action } => match action {
            UserCommands::List { limit } => {
                println!("Listing users (limit: {:?})", limit);
            }
            UserCommands::Create { email, name } => {
                println!("Creating user: {} <{}>", name, email);
            }
            UserCommands::Delete { id } => {
                println!("Deleting user: {}", id);
            }
        },
        Commands::Db { action } => match action {
            DbCommands::Migrate => println!("Running migrations"),
            DbCommands::Seed => println!("Seeding database"),
            DbCommands::Reset { force } => {
                if force {
                    println!("Force resetting database");
                } else {
                    println!("Use --force to reset");
                }
            }
        },
    }

    Ok(())
}
```

## TypedBuilder Advanced Usage

```rust
use typed_builder::TypedBuilder;
use std::time::Duration;

#[derive(Debug, TypedBuilder)]
pub struct HttpClientConfig {
    #[builder(setter(into))]
    base_url: String,

    #[builder(default = Duration::from_secs(30))]
    timeout: Duration,

    #[builder(default = Duration::from_secs(10))]
    connect_timeout: Duration,

    #[builder(default = 3)]
    max_retries: u32,

    #[builder(default, setter(strip_option, into))]
    user_agent: Option<String>,

    #[builder(default, setter(strip_option, into))]
    proxy: Option<String>,

    #[builder(default = true)]
    verify_ssl: bool,

    #[builder(default)]
    headers: std::collections::HashMap<String, String>,
}

impl HttpClientConfig {
    pub fn with_header(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.headers.insert(key.into(), value.into());
        self
    }
}

// Usage
let config = HttpClientConfig::builder()
    .base_url("https://api.example.com")
    .timeout(Duration::from_secs(60))
    .max_retries(5)
    .user_agent("MyApp/1.0")
    .build()
    .with_header("X-Custom", "value");
```

## derive_more Complete Examples

```rust
use derive_more::{
    Add, AddAssign, Constructor, Deref, DerefMut, Display,
    From, Into, Index, IndexMut, IntoIterator, Mul, Not, Sum,
};

// Display derive
#[derive(Debug, Display)]
pub enum LogLevel {
    #[display("DEBUG")]
    Debug,
    #[display("INFO")]
    Info,
    #[display("WARN")]
    Warn,
    #[display("ERROR")]
    Error,
}

// From/Into derives
#[derive(Debug, Clone, From, Into, Deref, DerefMut)]
pub struct Email(String);

#[derive(Debug, Clone, From, Into)]
pub struct UserId(uuid::Uuid);

// Constructor derive
#[derive(Debug, Constructor)]
pub struct Point {
    x: f64,
    y: f64,
}

// Arithmetic derives
#[derive(Debug, Clone, Copy, Add, AddAssign, Mul, Sum)]
pub struct Money(i64);

// Index derives
#[derive(Debug, Index, IndexMut, IntoIterator)]
pub struct Matrix {
    #[index]
    #[index_mut]
    #[into_iterator(owned, ref, ref_mut)]
    data: Vec<Vec<f64>>,
}

// Complex display
#[derive(Debug, Display)]
#[display("User {{ id: {id}, email: {email}, active: {active} }}")]
pub struct User {
    id: UserId,
    email: Email,
    active: bool,
}
```

## Utilities Checklist

```markdown
### Utilities Implementation Verification
- [ ] JWT service with access/refresh token pair
- [ ] Token verification with proper error handling
- [ ] Random generation utilities (alphanumeric, UUID, OTP)
- [ ] CLI with clap 4.0+ derive macros
- [ ] Subcommands properly organized
- [ ] TypedBuilder for complex configurations
- [ ] derive_more for common trait implementations
- [ ] Secure random using appropriate RNG
- [ ] Time-based UUIDs (v7) for sortable IDs
```

---
*Source: .cursor/rules/rust/features/utilities.mdc*
