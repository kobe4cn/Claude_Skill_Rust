---
description: >-
  Use when implementing HTTP clients in Rust. Covers reqwest with rustls-tls,
  compression, timeout configuration, error handling, retry logic, and authentication.
  Triggers: reqwest, HTTP client, REST client, API integration, HTTP request.
---

# Rust HTTP Client

## Quick Reference

Modern HTTP client patterns using reqwest with proper configuration and error handling.

## Reqwest Configuration

```toml
[dependencies]
reqwest = { version = "0.12", default-features = false, features = [
    "rustls-tls-webpki-roots",
    "http2",
    "json",
    "gzip",
    "brotli",
    "deflate"
] }
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.45", features = ["macros", "rt-multi-thread"] }
thiserror = "2.0"
url = "2.5"
```

## HTTP Client Builder

```rust
use reqwest::{Client, ClientBuilder, Response};
use std::time::Duration;
use url::Url;

pub struct HttpClient {
    client: Client,
    base_url: Url,
    timeout: Duration,
}

impl HttpClient {
    pub fn builder() -> HttpClientBuilder {
        HttpClientBuilder::new()
    }

    pub async fn get<T>(&self, path: &str) -> Result<T, HttpError>
    where
        T: for<'de> serde::Deserialize<'de>,
    {
        let url = self.base_url.join(path)?;
        let response = self.client
            .get(url)
            .timeout(self.timeout)
            .send()
            .await?;

        self.handle_response(response).await
    }

    pub async fn post<T, B>(&self, path: &str, body: &B) -> Result<T, HttpError>
    where
        T: for<'de> serde::Deserialize<'de>,
        B: serde::Serialize,
    {
        let url = self.base_url.join(path)?;
        let response = self.client
            .post(url)
            .json(body)
            .timeout(self.timeout)
            .send()
            .await?;

        self.handle_response(response).await
    }

    async fn handle_response<T>(&self, response: Response) -> Result<T, HttpError>
    where
        T: for<'de> serde::Deserialize<'de>,
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

pub struct HttpClientBuilder {
    base_url: Option<String>,
    timeout: Duration,
    user_agent: String,
}

impl HttpClientBuilder {
    pub fn new() -> Self {
        Self {
            base_url: None,
            timeout: Duration::from_secs(30),
            user_agent: "rust-http-client/1.0".to_string(),
        }
    }

    pub fn base_url(mut self, url: &str) -> Self {
        self.base_url = Some(url.to_string());
        self
    }

    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = timeout;
        self
    }

    pub fn build(self) -> Result<HttpClient, HttpError> {
        let base_url = self.base_url
            .ok_or_else(|| HttpError::Configuration("Base URL required".into()))?;

        let client = ClientBuilder::new()
            .timeout(self.timeout)
            .user_agent(&self.user_agent)
            .build()?;

        Ok(HttpClient {
            client,
            base_url: Url::parse(&base_url)?,
            timeout: self.timeout,
        })
    }
}
```

## Error Handling

```rust
#[derive(thiserror::Error, Debug)]
pub enum HttpError {
    #[error("HTTP request error: {0}")]
    Request(#[from] reqwest::Error),

    #[error("URL parsing error: {0}")]
    UrlParse(#[from] url::ParseError),

    #[error("Deserialization error: {error}, body: {body}")]
    Deserialization { error: String, body: String },

    #[error("Unexpected HTTP status {status}: {body}")]
    UnexpectedStatus { status: u16, body: String },

    #[error("Configuration error: {0}")]
    Configuration(String),
}

impl HttpError {
    pub fn is_retryable(&self) -> bool {
        matches!(
            self,
            HttpError::Request(_)
                | HttpError::UnexpectedStatus { status: 502..=504, .. }
        )
    }
}
```

## See Also

- `reference.md` - Complete patterns, retry logic, authentication
- `../api-design/` - API client design patterns
