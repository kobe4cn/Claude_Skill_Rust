# Rust HTTP Client Reference

## Complete HTTP Client Implementation

```rust
use reqwest::{Client, ClientBuilder, Response, StatusCode};
use serde::{Deserialize, Serialize};
use std::time::Duration;
use url::Url;

pub struct HttpClient {
    client: Client,
    base_url: Url,
    default_timeout: Duration,
}

impl HttpClient {
    pub fn builder() -> HttpClientBuilder {
        HttpClientBuilder::new()
    }

    pub async fn get<T>(&self, path: &str) -> Result<T, HttpError>
    where
        T: for<'de> Deserialize<'de>,
    {
        let url = self.base_url.join(path)?;
        let response = self.client
            .get(url)
            .timeout(self.default_timeout)
            .send()
            .await?;

        self.handle_response(response).await
    }

    pub async fn post<T, B>(&self, path: &str, body: &B) -> Result<T, HttpError>
    where
        T: for<'de> Deserialize<'de>,
        B: Serialize,
    {
        let url = self.base_url.join(path)?;
        let response = self.client
            .post(url)
            .json(body)
            .timeout(self.default_timeout)
            .send()
            .await?;

        self.handle_response(response).await
    }

    pub async fn put<T, B>(&self, path: &str, body: &B) -> Result<T, HttpError>
    where
        T: for<'de> Deserialize<'de>,
        B: Serialize,
    {
        let url = self.base_url.join(path)?;
        let response = self.client
            .put(url)
            .json(body)
            .timeout(self.default_timeout)
            .send()
            .await?;

        self.handle_response(response).await
    }

    pub async fn delete(&self, path: &str) -> Result<(), HttpError> {
        let url = self.base_url.join(path)?;
        let response = self.client
            .delete(url)
            .timeout(self.default_timeout)
            .send()
            .await?;

        if response.status().is_success() {
            Ok(())
        } else {
            let body = response.text().await.unwrap_or_default();
            Err(HttpError::UnexpectedStatus {
                status: response.status().as_u16(),
                body,
            })
        }
    }

    async fn handle_response<T>(&self, response: Response) -> Result<T, HttpError>
    where
        T: for<'de> Deserialize<'de>,
    {
        let status = response.status();

        if status.is_success() {
            let text = response.text().await?;
            serde_json::from_str(&text).map_err(|e| HttpError::Deserialization {
                error: e.to_string(),
                body: text,
            })
        } else {
            let body = response.text().await.unwrap_or_default();
            Err(HttpError::UnexpectedStatus {
                status: status.as_u16(),
                body,
            })
        }
    }
}
```

## HTTP Client Builder

```rust
use std::collections::HashMap;

pub struct HttpClientBuilder {
    base_url: Option<String>,
    timeout: Option<Duration>,
    connect_timeout: Option<Duration>,
    user_agent: Option<String>,
    default_headers: HashMap<String, String>,
    accept_invalid_certs: bool,
}

impl HttpClientBuilder {
    pub fn new() -> Self {
        Self {
            base_url: None,
            timeout: Some(Duration::from_secs(30)),
            connect_timeout: Some(Duration::from_secs(10)),
            user_agent: Some("rust-http-client/1.0".to_string()),
            default_headers: HashMap::new(),
            accept_invalid_certs: false,
        }
    }

    pub fn base_url(mut self, url: &str) -> Self {
        self.base_url = Some(url.to_string());
        self
    }

    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = Some(timeout);
        self
    }

    pub fn connect_timeout(mut self, timeout: Duration) -> Self {
        self.connect_timeout = Some(timeout);
        self
    }

    pub fn user_agent(mut self, agent: &str) -> Self {
        self.user_agent = Some(agent.to_string());
        self
    }

    pub fn header(mut self, key: &str, value: &str) -> Self {
        self.default_headers.insert(key.to_string(), value.to_string());
        self
    }

    pub fn bearer_token(self, token: &str) -> Self {
        self.header("Authorization", &format!("Bearer {}", token))
    }

    pub fn build(self) -> Result<HttpClient, HttpError> {
        let base_url = self.base_url
            .ok_or_else(|| HttpError::Configuration("Base URL is required".to_string()))?;

        let mut client_builder = ClientBuilder::new()
            .danger_accept_invalid_certs(self.accept_invalid_certs);

        if let Some(timeout) = self.timeout {
            client_builder = client_builder.timeout(timeout);
        }

        if let Some(connect_timeout) = self.connect_timeout {
            client_builder = client_builder.connect_timeout(connect_timeout);
        }

        if let Some(user_agent) = &self.user_agent {
            client_builder = client_builder.user_agent(user_agent);
        }

        // Add default headers
        let mut headers = reqwest::header::HeaderMap::new();
        for (key, value) in &self.default_headers {
            headers.insert(
                reqwest::header::HeaderName::from_bytes(key.as_bytes())
                    .map_err(|e| HttpError::Configuration(e.to_string()))?,
                value.parse().map_err(|e: reqwest::header::InvalidHeaderValue| {
                    HttpError::Configuration(e.to_string())
                })?,
            );
        }
        client_builder = client_builder.default_headers(headers);

        let client = client_builder.build()?;
        let parsed_url = Url::parse(&base_url)?;

        Ok(HttpClient {
            client,
            base_url: parsed_url,
            default_timeout: self.timeout.unwrap_or(Duration::from_secs(30)),
        })
    }
}
```

## Comprehensive Error Types

```rust
#[derive(thiserror::Error, Debug)]
pub enum HttpError {
    #[error("HTTP request error: {0}")]
    Request(#[from] reqwest::Error),

    #[error("URL parsing error: {0}")]
    UrlParse(#[from] url::ParseError),

    #[error("JSON serialization error: {0}")]
    Serialization(#[from] serde_json::Error),

    #[error("Deserialization error: {error}, body: {body}")]
    Deserialization { error: String, body: String },

    #[error("Unexpected HTTP status {status}: {body}")]
    UnexpectedStatus { status: u16, body: String },

    #[error("Configuration error: {0}")]
    Configuration(String),

    #[error("Timeout occurred")]
    Timeout,

    #[error("Authentication failed")]
    Authentication,

    #[error("Rate limit exceeded, retry after {retry_after} seconds")]
    RateLimit { retry_after: u64 },
}

impl HttpError {
    pub fn is_retryable(&self) -> bool {
        matches!(
            self,
            HttpError::Timeout
                | HttpError::RateLimit { .. }
                | HttpError::UnexpectedStatus { status: 502..=504, .. }
        )
    }

    pub fn retry_after(&self) -> Option<Duration> {
        match self {
            HttpError::RateLimit { retry_after } => Some(Duration::from_secs(*retry_after)),
            _ => None,
        }
    }
}
```

## Retry Logic

```rust
use tokio::time::sleep;

pub struct RetryConfig {
    pub max_retries: u32,
    pub initial_delay: Duration,
    pub max_delay: Duration,
    pub exponential_base: u32,
}

impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            max_retries: 3,
            initial_delay: Duration::from_millis(100),
            max_delay: Duration::from_secs(10),
            exponential_base: 2,
        }
    }
}

pub async fn with_retry<T, F, Fut>(
    config: &RetryConfig,
    operation: F,
) -> Result<T, HttpError>
where
    F: Fn() -> Fut,
    Fut: std::future::Future<Output = Result<T, HttpError>>,
{
    let mut attempts = 0;
    let mut delay = config.initial_delay;

    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(err) if err.is_retryable() && attempts < config.max_retries => {
                attempts += 1;

                // Check for rate limit specific delay
                if let Some(retry_after) = err.retry_after() {
                    sleep(retry_after).await;
                } else {
                    sleep(delay).await;
                    delay = std::cmp::min(
                        delay * config.exponential_base,
                        config.max_delay,
                    );
                }

                tracing::warn!(
                    attempt = attempts,
                    max_retries = config.max_retries,
                    "Retrying failed request"
                );
            }
            Err(err) => return Err(err),
        }
    }
}

// Usage
let result = with_retry(&RetryConfig::default(), || async {
    client.get::<UserResponse>("/users/123").await
}).await?;
```

## Authentication Patterns

```rust
pub enum AuthMethod {
    Bearer(String),
    ApiKey { key: String, header: String },
    Basic { username: String, password: String },
}

impl HttpClientBuilder {
    pub fn with_auth(self, auth: AuthMethod) -> Self {
        match auth {
            AuthMethod::Bearer(token) => {
                self.header("Authorization", &format!("Bearer {}", token))
            }
            AuthMethod::ApiKey { key, header } => {
                self.header(&header, &key)
            }
            AuthMethod::Basic { username, password } => {
                let encoded = base64::encode(format!("{}:{}", username, password));
                self.header("Authorization", &format!("Basic {}", encoded))
            }
        }
    }
}
```

## Response Types

```rust
#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ApiResponse<T> {
    pub data: T,
    pub meta: Option<ResponseMeta>,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ResponseMeta {
    pub total_count: Option<u64>,
    pub page: Option<u32>,
    pub per_page: Option<u32>,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ErrorResponse {
    pub error: ErrorDetails,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ErrorDetails {
    pub code: String,
    pub message: String,
    pub details: Option<serde_json::Value>,
}
```

## HTTP Client Checklist

```markdown
### HTTP Client Implementation Verification
- [ ] Uses reqwest with rustls-tls (not native-tls)
- [ ] Compression features enabled (gzip, brotli, deflate)
- [ ] Proper timeout configuration
- [ ] User-Agent header configured
- [ ] Structured error handling with retryable errors
- [ ] Retry logic with exponential backoff
- [ ] Authentication patterns implemented
- [ ] Response types with camelCase deserialization
- [ ] Base URL configuration pattern
- [ ] Proper status code handling
```

---
*Source: .cursor/rules/rust/features/http-client.mdc*
