# Rust 配置管理完整参考

## 完整配置结构

```rust
use figment::{Figment, providers::{Format, Yaml, Toml, Json, Env, Serialized}};
use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, Validate)]
#[serde(rename_all = "snake_case")]
pub struct AppConfig {
    #[validate(length(min = 1, max = 100))]
    pub name: String,

    #[serde(default = "default_environment")]
    pub environment: Environment,

    #[validate(nested)]
    pub server: ServerConfig,

    #[validate(nested)]
    pub database: DatabaseConfig,

    #[validate(nested)]
    pub logging: LoggingConfig,

    #[serde(default)]
    pub features: FeatureFlags,
}

#[derive(Debug, Clone, Serialize, Deserialize, Validate)]
pub struct ServerConfig {
    #[serde(default = "default_host")]
    pub host: String,

    #[validate(range(min = 1, max = 65535))]
    pub port: u16,

    #[serde(default = "default_workers")]
    pub workers: usize,

    #[serde(default)]
    pub tls: Option<TlsConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize, Validate)]
pub struct DatabaseConfig {
    #[validate(url)]
    pub url: String,

    #[validate(range(min = 1, max = 100))]
    #[serde(default = "default_pool_size")]
    pub pool_size: u32,

    #[serde(default = "default_timeout")]
    pub connect_timeout_secs: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LoggingConfig {
    #[serde(default = "default_log_level")]
    pub level: String,

    #[serde(default)]
    pub json_format: bool,
}

#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct FeatureFlags {
    #[serde(default)]
    pub enable_metrics: bool,

    #[serde(default)]
    pub enable_tracing: bool,

    #[serde(default)]
    pub experimental_features: bool,
}

// 默认值函数
fn default_host() -> String { "0.0.0.0".to_string() }
fn default_workers() -> usize { num_cpus::get() }
fn default_pool_size() -> u32 { 10 }
fn default_timeout() -> u64 { 30 }
fn default_log_level() -> String { "info".to_string() }
fn default_environment() -> Environment { Environment::Development }
```

## 环境类型

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Environment {
    Development,
    Testing,
    Staging,
    Production,
}

impl Environment {
    pub fn from_env() -> Self {
        std::env::var("APP_ENV")
            .or_else(|_| std::env::var("ENVIRONMENT"))
            .map(|s| match s.to_lowercase().as_str() {
                "prod" | "production" => Self::Production,
                "staging" => Self::Staging,
                "test" | "testing" => Self::Testing,
                _ => Self::Development,
            })
            .unwrap_or(Self::Development)
    }

    pub fn is_production(&self) -> bool {
        matches!(self, Self::Production)
    }

    pub fn config_file(&self) -> &str {
        match self {
            Self::Development => "config/development.yaml",
            Self::Testing => "config/testing.yaml",
            Self::Staging => "config/staging.yaml",
            Self::Production => "config/production.yaml",
        }
    }
}
```

## 多格式加载

```rust
use figment::providers::Env;

impl AppConfig {
    /// 从多个来源加载配置
    pub fn load() -> Result<Self, ConfigError> {
        let env = Environment::from_env();

        let config: Self = Figment::new()
            // 1. 默认值
            .merge(Serialized::defaults(Self::default()))
            // 2. 基础配置文件
            .merge(Yaml::file("config/base.yaml"))
            // 3. 环境特定配置
            .merge(Yaml::file(env.config_file()))
            // 4. 本地覆盖（不提交到版本控制）
            .merge(Yaml::file("config/local.yaml"))
            // 5. 环境变量覆盖
            .merge(Env::prefixed("APP_").split("__"))
            .extract()
            .map_err(ConfigError::Parse)?;

        // 验证配置
        config.validate().map_err(ConfigError::Validation)?;

        Ok(config)
    }

    /// 从特定文件加载
    pub fn from_file(path: &str) -> Result<Self, ConfigError> {
        let figment = match path.rsplit('.').next() {
            Some("yaml") | Some("yml") => Figment::new().merge(Yaml::file(path)),
            Some("toml") => Figment::new().merge(Toml::file(path)),
            Some("json") => Figment::new().merge(Json::file(path)),
            _ => return Err(ConfigError::UnsupportedFormat(path.to_string())),
        };

        let config: Self = figment.extract().map_err(ConfigError::Parse)?;
        config.validate().map_err(ConfigError::Validation)?;
        Ok(config)
    }
}

impl Default for AppConfig {
    fn default() -> Self {
        Self {
            name: "app".to_string(),
            environment: Environment::Development,
            server: ServerConfig {
                host: "0.0.0.0".to_string(),
                port: 8080,
                workers: num_cpus::get(),
                tls: None,
            },
            database: DatabaseConfig {
                url: "postgres://localhost/app".to_string(),
                pool_size: 10,
                connect_timeout_secs: 30,
            },
            logging: LoggingConfig {
                level: "info".to_string(),
                json_format: false,
            },
            features: FeatureFlags::default(),
        }
    }
}
```

## 热重载实现

```rust
use arc_swap::ArcSwap;
use notify::{RecommendedWatcher, RecursiveMode, Watcher, Event};
use std::sync::Arc;
use tokio::sync::broadcast;

pub struct ConfigManager<T> {
    current: Arc<ArcSwap<T>>,
    reload_tx: broadcast::Sender<Arc<T>>,
    _watcher: Option<RecommendedWatcher>,
}

impl<T: Clone + Send + Sync + 'static> ConfigManager<T> {
    pub fn new(initial: T) -> Self {
        let (reload_tx, _) = broadcast::channel(16);
        Self {
            current: Arc::new(ArcSwap::from_pointee(initial)),
            reload_tx,
            _watcher: None,
        }
    }

    /// 获取当前配置
    pub fn get(&self) -> Arc<T> {
        self.current.load_full()
    }

    /// 更新配置
    pub fn update(&self, new_config: T) {
        let new = Arc::new(new_config);
        self.current.store(new.clone());
        let _ = self.reload_tx.send(new);
    }

    /// 订阅配置更新
    pub fn subscribe(&self) -> broadcast::Receiver<Arc<T>> {
        self.reload_tx.subscribe()
    }

    /// 启用文件监视
    pub fn with_file_watcher<F>(mut self, path: &str, reload_fn: F) -> Result<Self, notify::Error>
    where
        F: Fn() -> Option<T> + Send + 'static,
    {
        let current = self.current.clone();
        let reload_tx = self.reload_tx.clone();

        let mut watcher = notify::recommended_watcher(move |res: Result<Event, _>| {
            if let Ok(event) = res {
                if event.kind.is_modify() {
                    if let Some(new_config) = reload_fn() {
                        let new = Arc::new(new_config);
                        current.store(new.clone());
                        let _ = reload_tx.send(new);
                        tracing::info!("Configuration reloaded");
                    }
                }
            }
        })?;

        watcher.watch(std::path::Path::new(path), RecursiveMode::NonRecursive)?;
        self._watcher = Some(watcher);

        Ok(self)
    }
}

// 使用示例
pub async fn setup_config_manager() -> Result<ConfigManager<AppConfig>, ConfigError> {
    let config = AppConfig::load()?;

    let manager = ConfigManager::new(config)
        .with_file_watcher("config/", || {
            AppConfig::load().ok()
        })
        .map_err(|e| ConfigError::Watch(e.to_string()))?;

    Ok(manager)
}
```

## 响应配置更新的组件

```rust
use std::sync::Arc;
use tokio::sync::broadcast;

#[async_trait::async_trait]
pub trait ConfigAware: Send + Sync {
    type Config;

    async fn on_config_update(&self, config: Arc<Self::Config>);
}

pub struct DatabasePool {
    config: Arc<ArcSwap<DatabaseConfig>>,
}

#[async_trait::async_trait]
impl ConfigAware for DatabasePool {
    type Config = DatabaseConfig;

    async fn on_config_update(&self, config: Arc<Self::Config>) {
        self.config.store(config);
        // 可选：重建连接池
        tracing::info!("Database config updated");
    }
}

// 注册组件以接收更新
pub async fn register_config_listeners(
    manager: &ConfigManager<AppConfig>,
    components: Vec<Arc<dyn ConfigAware<Config = AppConfig>>>,
) {
    let mut rx = manager.subscribe();

    tokio::spawn(async move {
        while let Ok(config) = rx.recv().await {
            for component in &components {
                component.on_config_update(config.clone()).await;
            }
        }
    });
}
```

## 错误处理

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ConfigError {
    #[error("Failed to parse configuration: {0}")]
    Parse(#[from] figment::Error),

    #[error("Configuration validation failed: {0}")]
    Validation(#[from] validator::ValidationErrors),

    #[error("Unsupported configuration format: {0}")]
    UnsupportedFormat(String),

    #[error("Failed to watch configuration file: {0}")]
    Watch(String),

    #[error("Missing required configuration: {0}")]
    Missing(String),
}
```

## 敏感配置处理

```rust
use secrecy::{ExposeSecret, Secret};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SecretConfig {
    #[serde(serialize_with = "serialize_secret")]
    pub api_key: Secret<String>,

    #[serde(serialize_with = "serialize_secret")]
    pub database_password: Secret<String>,
}

fn serialize_secret<S>(_: &Secret<String>, serializer: S) -> Result<S::Ok, S::Error>
where
    S: serde::Serializer,
{
    serializer.serialize_str("***REDACTED***")
}

impl SecretConfig {
    pub fn api_key(&self) -> &str {
        self.api_key.expose_secret()
    }
}
```

## 测试

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::TempDir;

    #[test]
    fn test_default_config() {
        let config = AppConfig::default();
        assert_eq!(config.server.port, 8080);
        assert!(config.validate().is_ok());
    }

    #[test]
    fn test_env_override() {
        std::env::set_var("APP_SERVER__PORT", "9000");
        let config = AppConfig::load().unwrap();
        assert_eq!(config.server.port, 9000);
        std::env::remove_var("APP_SERVER__PORT");
    }

    #[test]
    fn test_validation_failure() {
        let mut config = AppConfig::default();
        config.server.port = 0; // 无效端口
        assert!(config.validate().is_err());
    }

    #[test]
    fn test_config_from_yaml() {
        let dir = TempDir::new().unwrap();
        let path = dir.path().join("test.yaml");
        std::fs::write(&path, r#"
            name: test-app
            server:
              port: 3000
        "#).unwrap();

        let config = AppConfig::from_file(path.to_str().unwrap()).unwrap();
        assert_eq!(config.name, "test-app");
        assert_eq!(config.server.port, 3000);
    }

    #[tokio::test]
    async fn test_config_manager() {
        let config = AppConfig::default();
        let manager = ConfigManager::new(config);

        let mut rx = manager.subscribe();

        let mut new_config = AppConfig::default();
        new_config.server.port = 9999;
        manager.update(new_config);

        let received = rx.recv().await.unwrap();
        assert_eq!(received.server.port, 9999);
    }
}
```

## 配置文件示例

```yaml
# config/base.yaml
name: my-service
server:
  host: 0.0.0.0
  port: 8080
  workers: 4
database:
  pool_size: 10
  connect_timeout_secs: 30
logging:
  level: info
  json_format: false

---
# config/production.yaml
server:
  workers: 16
  tls:
    cert_path: /etc/ssl/cert.pem
    key_path: /etc/ssl/key.pem
database:
  pool_size: 50
logging:
  level: warn
  json_format: true
features:
  enable_metrics: true
  enable_tracing: true
```

## 检查清单

```markdown
### 配置管理检查清单
- [ ] 使用 figment 支持多格式
- [ ] 所有配置都有 serde(default) 或默认值
- [ ] 使用 validator 验证配置
- [ ] 环境变量可覆盖所有配置
- [ ] 敏感配置使用 Secret<T>
- [ ] 配置更新不需要重启
- [ ] 配置验证错误信息清晰
- [ ] 开发/生产配置分离
- [ ] 本地配置不提交版本控制
```
