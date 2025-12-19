# Rust 错误处理完整参考

## 库错误类型完整实现

```rust
use thiserror::Error;
use std::fmt;

/// 主错误类型 - 库 crate
#[derive(Error, Debug)]
pub enum MyLibError {
    // 带结构化数据的错误
    #[error("Invalid input: {message}")]
    InvalidInput {
        message: String,
        field: Option<String>,
    },

    #[error("Resource not found: {resource_type} with id {id}")]
    NotFound {
        resource_type: &'static str,
        id: String,
    },

    #[error("Operation timed out after {duration_ms}ms")]
    Timeout { duration_ms: u64 },

    #[error("Rate limit exceeded: {limit} requests per {window_secs}s")]
    RateLimited { limit: u32, window_secs: u32 },

    // 自动转换的外部错误
    #[error("Database error")]
    Database(#[from] sqlx::Error),

    #[error("Serialization error")]
    Serialization(#[from] serde_json::Error),

    #[error("HTTP client error")]
    Http(#[from] reqwest::Error),

    #[error("IO error")]
    Io(#[from] std::io::Error),

    // 内部错误（不暴露细节）
    #[error("Internal error")]
    Internal(#[source] Box<dyn std::error::Error + Send + Sync>),
}

/// 库的 Result 类型别名
pub type Result<T> = std::result::Result<T, MyLibError>;
```

## 错误构造辅助方法

```rust
impl MyLibError {
    /// 创建输入验证错误
    pub fn invalid_input(msg: impl Into<String>) -> Self {
        Self::InvalidInput {
            message: msg.into(),
            field: None,
        }
    }

    /// 创建带字段的输入验证错误
    pub fn invalid_field(field: impl Into<String>, msg: impl Into<String>) -> Self {
        Self::InvalidInput {
            message: msg.into(),
            field: Some(field.into()),
        }
    }

    /// 创建资源未找到错误
    pub fn not_found(resource_type: &'static str, id: impl Into<String>) -> Self {
        Self::NotFound {
            resource_type,
            id: id.into(),
        }
    }

    /// 创建超时错误
    pub fn timeout(duration_ms: u64) -> Self {
        Self::Timeout { duration_ms }
    }

    /// 包装任意错误为内部错误
    pub fn internal<E>(err: E) -> Self
    where
        E: std::error::Error + Send + Sync + 'static,
    {
        Self::Internal(Box::new(err))
    }

    /// 检查是否为可重试错误
    pub fn is_retryable(&self) -> bool {
        matches!(
            self,
            Self::Timeout { .. }
                | Self::RateLimited { .. }
                | Self::Database(_)
                | Self::Http(_)
        )
    }

    /// 获取 HTTP 状态码
    pub fn status_code(&self) -> u16 {
        match self {
            Self::InvalidInput { .. } => 400,
            Self::NotFound { .. } => 404,
            Self::RateLimited { .. } => 429,
            Self::Timeout { .. } => 504,
            _ => 500,
        }
    }
}
```

## 二进制应用错误处理 (anyhow)

```rust
use anyhow::{Context, Result, bail, ensure, anyhow};

fn main() -> Result<()> {
    // 设置 panic hook 以获得更好的错误信息
    color_eyre::install()?;

    run().map_err(|e| {
        eprintln!("Error: {:?}", e);
        e
    })
}

fn run() -> Result<()> {
    let config = load_config()
        .context("Failed to load configuration")?;

    let db = connect_database(&config.database_url)
        .context("Failed to connect to database")?;

    process_data(&db)
        .context("Failed to process data")?;

    Ok(())
}

fn load_config() -> Result<Config> {
    let path = std::env::var("CONFIG_PATH")
        .context("CONFIG_PATH environment variable not set")?;

    let content = std::fs::read_to_string(&path)
        .with_context(|| format!("Failed to read config file: {}", path))?;

    let config: Config = toml::from_str(&content)
        .with_context(|| format!("Failed to parse config file: {}", path))?;

    // 验证配置
    ensure!(!config.database_url.is_empty(), "database_url cannot be empty");
    ensure!(config.port > 0, "port must be positive");

    if config.debug && config.environment == "production" {
        bail!("Debug mode cannot be enabled in production");
    }

    Ok(config)
}

fn connect_database(url: &str) -> Result<Database> {
    Database::connect(url)
        .map_err(|e| anyhow!("Database connection failed: {}", e))
}
```

## 错误链和上下文

```rust
use anyhow::{Context, Result};

/// 好的模式：添加有意义的上下文
fn process_user(user_id: &str) -> Result<User> {
    let raw_data = fetch_user_data(user_id)
        .with_context(|| format!("Failed to fetch user {}", user_id))?;

    let user = parse_user_data(&raw_data)
        .with_context(|| format!("Failed to parse user data for {}", user_id))?;

    validate_user(&user)
        .with_context(|| format!("User {} validation failed", user_id))?;

    Ok(user)
}

/// 好的模式：静态上下文（性能更好）
fn load_settings() -> Result<Settings> {
    let content = std::fs::read_to_string("settings.json")
        .context("Failed to read settings.json")?;

    serde_json::from_str(&content)
        .context("Failed to parse settings.json")
}

/// 不好的模式：没有上下文
fn bad_example() -> Result<Data> {
    let data = fetch_data()?;  // 哪里出错了？
    let parsed = parse_data(&data)?;  // 哪个解析失败了？
    Ok(parsed)
}
```

## HTTP API 错误响应

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde::Serialize;

#[derive(Debug, Serialize)]
pub struct ApiError {
    pub code: String,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub details: Option<serde_json::Value>,
}

impl IntoResponse for MyLibError {
    fn into_response(self) -> Response {
        let (status, code, message) = match &self {
            MyLibError::InvalidInput { message, field } => {
                let details = field.as_ref().map(|f| {
                    serde_json::json!({ "field": f })
                });
                return (
                    StatusCode::BAD_REQUEST,
                    Json(ApiError {
                        code: "INVALID_INPUT".to_string(),
                        message: message.clone(),
                        details,
                    }),
                ).into_response();
            }
            MyLibError::NotFound { resource_type, id } => (
                StatusCode::NOT_FOUND,
                "NOT_FOUND",
                format!("{} '{}' not found", resource_type, id),
            ),
            MyLibError::RateLimited { limit, window_secs } => (
                StatusCode::TOO_MANY_REQUESTS,
                "RATE_LIMITED",
                format!("Rate limit of {} requests per {}s exceeded", limit, window_secs),
            ),
            MyLibError::Timeout { duration_ms } => (
                StatusCode::GATEWAY_TIMEOUT,
                "TIMEOUT",
                format!("Operation timed out after {}ms", duration_ms),
            ),
            _ => {
                // 记录内部错误但不暴露细节
                tracing::error!("Internal error: {:?}", self);
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "INTERNAL_ERROR",
                    "An internal error occurred".to_string(),
                )
            }
        };

        (
            status,
            Json(ApiError {
                code: code.to_string(),
                message,
                details: None,
            }),
        ).into_response()
    }
}
```

## 错误分类

```rust
/// 错误严重程度
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ErrorSeverity {
    /// 用户错误，无需报警
    UserError,
    /// 临时性错误，可重试
    Transient,
    /// 系统错误，需要关注
    SystemError,
    /// 严重错误，需要立即处理
    Critical,
}

impl MyLibError {
    pub fn severity(&self) -> ErrorSeverity {
        match self {
            Self::InvalidInput { .. } | Self::NotFound { .. } => ErrorSeverity::UserError,
            Self::Timeout { .. } | Self::RateLimited { .. } => ErrorSeverity::Transient,
            Self::Database(_) | Self::Http(_) => ErrorSeverity::SystemError,
            Self::Internal(_) => ErrorSeverity::Critical,
            _ => ErrorSeverity::SystemError,
        }
    }

    pub fn should_alert(&self) -> bool {
        matches!(
            self.severity(),
            ErrorSeverity::SystemError | ErrorSeverity::Critical
        )
    }
}
```

## 自定义 Result 扩展

```rust
pub trait ResultExt<T> {
    /// 添加上下文并记录错误
    fn log_err(self, msg: &str) -> Result<T>;

    /// 转换为可选值，记录错误
    fn ok_or_log(self) -> Option<T>;
}

impl<T, E: std::fmt::Debug> ResultExt<T> for std::result::Result<T, E> {
    fn log_err(self, msg: &str) -> Result<T> {
        self.map_err(|e| {
            tracing::error!("{}: {:?}", msg, e);
            anyhow::anyhow!("{}: {:?}", msg, e)
        })
    }

    fn ok_or_log(self) -> Option<T> {
        match self {
            Ok(v) => Some(v),
            Err(e) => {
                tracing::warn!("Operation failed: {:?}", e);
                None
            }
        }
    }
}

// 使用示例
fn example() -> Result<()> {
    let data = fetch_data()
        .log_err("Failed to fetch data")?;

    // 可选操作，失败时继续
    let extra = fetch_optional_data().ok_or_log();

    Ok(())
}
```

## 测试错误

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_error_display() {
        let err = MyLibError::not_found("User", "123");
        assert_eq!(
            err.to_string(),
            "Resource not found: User with id 123"
        );
    }

    #[test]
    fn test_error_status_code() {
        assert_eq!(
            MyLibError::invalid_input("bad input").status_code(),
            400
        );
        assert_eq!(
            MyLibError::not_found("User", "1").status_code(),
            404
        );
    }

    #[test]
    fn test_error_is_retryable() {
        assert!(!MyLibError::invalid_input("bad").is_retryable());
        assert!(MyLibError::timeout(1000).is_retryable());
    }

    #[test]
    fn test_error_from_conversion() {
        let io_err = std::io::Error::new(
            std::io::ErrorKind::NotFound,
            "file not found"
        );
        let lib_err: MyLibError = io_err.into();
        assert!(matches!(lib_err, MyLibError::Io(_)));
    }

    #[test]
    fn test_anyhow_context() {
        fn failing_op() -> Result<()> {
            Err(anyhow::anyhow!("root cause"))
        }

        let result = failing_op().context("operation failed");
        assert!(result.is_err());

        let err = result.unwrap_err();
        assert!(err.to_string().contains("operation failed"));

        // 检查错误链
        let chain: Vec<_> = err.chain().collect();
        assert_eq!(chain.len(), 2);
    }
}
```

## 检查清单

```markdown
### 库 Crate 错误检查清单
- [ ] 错误定义在集中的 errors.rs 文件
- [ ] 所有错误类型 derive Error, Debug
- [ ] 使用 #[error] 提供有意义的消息
- [ ] 使用 #[from] 自动转换外部错误
- [ ] 提供错误构造辅助方法
- [ ] 实现 status_code() 用于 HTTP 响应
- [ ] 实现 is_retryable() 用于重试逻辑
- [ ] 定义 Result<T> 类型别名

### 二进制 Crate 错误检查清单
- [ ] 使用 anyhow::Result 作为返回类型
- [ ] 所有 ? 操作都添加 context
- [ ] 使用 with_context 添加动态上下文
- [ ] 使用 ensure! 进行前置条件检查
- [ ] 使用 bail! 进行早期返回
- [ ] 用户友好的错误消息
- [ ] 致命错误时正确清理资源
```
