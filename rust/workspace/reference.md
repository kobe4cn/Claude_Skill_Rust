# Rust 工作空间组织完整参考

## 完整工作空间 Cargo.toml

```toml
[workspace]
resolver = "2"
members = [
    # 共享基础设施
    "shared/shared-types",
    "shared/shared-config",
    "shared/shared-observability",
    "shared/shared-testing",

    # 用户管理子系统
    "services/user-management/user-core",
    "services/user-management/user-api",
    "services/user-management/user-worker",

    # 订单管理子系统
    "services/order-management/order-core",
    "services/order-management/order-api",
    "services/order-management/order-processor",

    # 平台基础设施
    "platform/api-gateway",
    "platform/admin-cli",
]

# 排除示例和工具目录
exclude = [
    "examples",
    "tools",
]

[workspace.package]
edition = "2024"
version = "0.1.0"
authors = ["Your Team <team@example.com>"]
license = "MIT"
repository = "https://github.com/org/project"

[workspace.dependencies]
# 异步运行时
tokio = { version = "1.45", features = ["full"] }
tokio-util = "0.7"

# 序列化
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Web 框架
axum = { version = "0.8", features = ["macros"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace", "compression-gzip"] }

# 数据库
sqlx = { version = "0.8", features = ["postgres", "runtime-tokio", "tls-rustls", "chrono", "uuid"] }

# gRPC
tonic = { version = "0.13", features = ["gzip", "tls"] }
prost = "0.13"

# 配置
figment = { version = "0.10", features = ["toml", "yaml", "env"] }
arc-swap = "1.0"

# 可观测性
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
prometheus = "0.13"
opentelemetry = "0.22"

# 错误处理
thiserror = "1.0"
anyhow = "1.0"

# 工具
uuid = { version = "1.0", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }

# 内部 crate
shared-types = { path = "shared/shared-types" }
shared-config = { path = "shared/shared-config" }
shared-observability = { path = "shared/shared-observability" }
user-core = { path = "services/user-management/user-core" }
order-core = { path = "services/order-management/order-core" }

[workspace.lints.rust]
unsafe_code = "forbid"
missing_docs = "warn"

[workspace.lints.clippy]
all = "warn"
pedantic = "warn"
nursery = "warn"

# 发布配置
[profile.release]
lto = true
codegen-units = 1
panic = "abort"
strip = true

[profile.dev]
opt-level = 0
debug = true

[profile.dev.package."*"]
opt-level = 3
```

## 子 Crate Cargo.toml 模板

```toml
# services/user-management/user-api/Cargo.toml
[package]
name = "user-api"
version.workspace = true
edition.workspace = true
authors.workspace = true
license.workspace = true

[dependencies]
# 内部依赖
user-core = { workspace = true }
shared-types = { workspace = true }
shared-config = { workspace = true }
shared-observability = { workspace = true }

# 外部依赖（使用 workspace 版本）
tokio = { workspace = true }
axum = { workspace = true }
tower = { workspace = true }
tower-http = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
tracing = { workspace = true }
anyhow = { workspace = true }

[dev-dependencies]
tokio-test = "0.4"
axum-test = "0.1"

[lints]
workspace = true
```

## 共享类型 Crate

```rust
// shared/shared-types/src/lib.rs
pub mod ids;
pub mod events;
pub mod pagination;

pub use ids::*;
pub use events::*;
pub use pagination::*;
```

```rust
// shared/shared-types/src/ids.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use std::fmt;

macro_rules! define_id {
    ($name:ident) => {
        #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
        #[serde(transparent)]
        pub struct $name(pub Uuid);

        impl $name {
            pub fn new() -> Self {
                Self(Uuid::new_v4())
            }

            pub fn from_uuid(uuid: Uuid) -> Self {
                Self(uuid)
            }

            pub fn as_uuid(&self) -> &Uuid {
                &self.0
            }
        }

        impl Default for $name {
            fn default() -> Self {
                Self::new()
            }
        }

        impl fmt::Display for $name {
            fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
                write!(f, "{}", self.0)
            }
        }

        impl std::str::FromStr for $name {
            type Err = uuid::Error;

            fn from_str(s: &str) -> Result<Self, Self::Err> {
                Ok(Self(Uuid::parse_str(s)?))
            }
        }
    };
}

define_id!(UserId);
define_id!(OrderId);
define_id!(ProductId);
define_id!(TenantId);
```

```rust
// shared/shared-types/src/events.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DomainEvent<T> {
    pub id: Uuid,
    pub event_type: String,
    pub timestamp: DateTime<Utc>,
    pub aggregate_id: String,
    pub payload: T,
    pub metadata: EventMetadata,
}

#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct EventMetadata {
    pub correlation_id: Option<String>,
    pub causation_id: Option<String>,
    pub user_id: Option<String>,
}

impl<T> DomainEvent<T> {
    pub fn new(event_type: impl Into<String>, aggregate_id: impl Into<String>, payload: T) -> Self {
        Self {
            id: Uuid::new_v4(),
            event_type: event_type.into(),
            timestamp: Utc::now(),
            aggregate_id: aggregate_id.into(),
            payload,
            metadata: EventMetadata::default(),
        }
    }

    pub fn with_metadata(mut self, metadata: EventMetadata) -> Self {
        self.metadata = metadata;
        self
    }
}
```

## 子系统核心 Crate 模式

```rust
// services/user-management/user-core/src/lib.rs
pub mod domain;
pub mod repository;
pub mod service;
pub mod errors;

pub use domain::*;
pub use repository::UserRepository;
pub use service::UserService;
pub use errors::UserError;
```

```rust
// services/user-management/user-core/src/domain.rs
use chrono::{DateTime, Utc};
use shared_types::UserId;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: UserId,
    pub email: String,
    pub username: String,
    pub display_name: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct CreateUser {
    pub email: String,
    pub username: String,
    pub display_name: Option<String>,
}

#[derive(Debug, Clone, Default)]
pub struct UpdateUser {
    pub email: Option<String>,
    pub username: Option<String>,
    pub display_name: Option<Option<String>>,
}
```

```rust
// services/user-management/user-core/src/repository.rs
use crate::{User, CreateUser, UpdateUser, UserError};
use shared_types::UserId;
use async_trait::async_trait;

#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: UserId) -> Result<Option<User>, UserError>;
    async fn find_by_email(&self, email: &str) -> Result<Option<User>, UserError>;
    async fn create(&self, user: CreateUser) -> Result<User, UserError>;
    async fn update(&self, id: UserId, update: UpdateUser) -> Result<User, UserError>;
    async fn delete(&self, id: UserId) -> Result<(), UserError>;
    async fn list(&self, offset: i64, limit: i64) -> Result<Vec<User>, UserError>;
}
```

```rust
// services/user-management/user-core/src/service.rs
use crate::{User, CreateUser, UpdateUser, UserError, UserRepository};
use shared_types::UserId;
use std::sync::Arc;

pub struct UserService<R: UserRepository> {
    repo: Arc<R>,
}

impl<R: UserRepository> UserService<R> {
    pub fn new(repo: Arc<R>) -> Self {
        Self { repo }
    }

    pub async fn get_user(&self, id: UserId) -> Result<User, UserError> {
        self.repo
            .find_by_id(id)
            .await?
            .ok_or_else(|| UserError::not_found(id))
    }

    pub async fn create_user(&self, input: CreateUser) -> Result<User, UserError> {
        // 检查邮箱是否已存在
        if self.repo.find_by_email(&input.email).await?.is_some() {
            return Err(UserError::email_exists(&input.email));
        }

        self.repo.create(input).await
    }

    pub async fn update_user(&self, id: UserId, update: UpdateUser) -> Result<User, UserError> {
        // 确保用户存在
        self.get_user(id).await?;

        // 如果更新邮箱，检查新邮箱是否已被使用
        if let Some(ref email) = update.email {
            if let Some(existing) = self.repo.find_by_email(email).await? {
                if existing.id != id {
                    return Err(UserError::email_exists(email));
                }
            }
        }

        self.repo.update(id, update).await
    }
}
```

## 子系统间通信

```rust
// shared/shared-types/src/service_client.rs
use async_trait::async_trait;

/// 用户服务客户端 trait - 供其他子系统使用
#[async_trait]
pub trait UserServiceClient: Send + Sync {
    async fn get_user(&self, id: UserId) -> Result<UserInfo, ServiceError>;
    async fn validate_user(&self, id: UserId) -> Result<bool, ServiceError>;
}

/// 跨服务传递的用户信息（最小化）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserInfo {
    pub id: UserId,
    pub username: String,
    pub email: String,
}

/// gRPC 实现
pub struct GrpcUserClient {
    client: user_grpc::UserServiceClient<Channel>,
}

#[async_trait]
impl UserServiceClient for GrpcUserClient {
    async fn get_user(&self, id: UserId) -> Result<UserInfo, ServiceError> {
        let request = user_grpc::GetUserRequest {
            id: id.to_string(),
        };

        let response = self.client
            .clone()
            .get_user(request)
            .await
            .map_err(|e| ServiceError::Rpc(e.to_string()))?;

        let user = response.into_inner();
        Ok(UserInfo {
            id: user.id.parse()?,
            username: user.username,
            email: user.email,
        })
    }
}
```

## 服务组件管理

```rust
use arc_swap::ArcSwap;
use std::sync::Arc;
use tokio::sync::broadcast;

pub struct ServiceManager<C> {
    config: Arc<ArcSwap<C>>,
    components: Vec<Arc<dyn ServiceComponent>>,
    shutdown_tx: broadcast::Sender<()>,
}

#[async_trait::async_trait]
pub trait ServiceComponent: Send + Sync {
    fn name(&self) -> &str;
    async fn start(&self) -> Result<(), anyhow::Error>;
    async fn stop(&self) -> Result<(), anyhow::Error>;
    fn on_config_update(&self, config: &dyn std::any::Any);
    async fn health_check(&self) -> HealthStatus;
}

impl<C: Send + Sync + 'static> ServiceManager<C> {
    pub fn new(config: C) -> Self {
        let (shutdown_tx, _) = broadcast::channel(1);
        Self {
            config: Arc::new(ArcSwap::from_pointee(config)),
            components: Vec::new(),
            shutdown_tx,
        }
    }

    pub fn register<T: ServiceComponent + 'static>(&mut self, component: T) {
        self.components.push(Arc::new(component));
    }

    pub async fn start_all(&self) -> Result<(), anyhow::Error> {
        for component in &self.components {
            tracing::info!("Starting component: {}", component.name());
            component.start().await?;
        }
        Ok(())
    }

    pub async fn stop_all(&self) -> Result<(), anyhow::Error> {
        let _ = self.shutdown_tx.send(());

        for component in self.components.iter().rev() {
            tracing::info!("Stopping component: {}", component.name());
            if let Err(e) = component.stop().await {
                tracing::error!("Error stopping {}: {}", component.name(), e);
            }
        }
        Ok(())
    }

    pub fn update_config(&self, new_config: C) {
        let new = Arc::new(new_config);
        self.config.store(new.clone());

        for component in &self.components {
            component.on_config_update(new.as_ref() as &dyn std::any::Any);
        }
    }

    pub fn get_config(&self) -> Arc<C> {
        self.config.load_full()
    }

    pub fn subscribe_shutdown(&self) -> broadcast::Receiver<()> {
        self.shutdown_tx.subscribe()
    }
}
```

## 构建脚本模式

```rust
// build.rs for workspace root
fn main() {
    // 设置构建时间
    println!(
        "cargo:rustc-env=BUILD_TIMESTAMP={}",
        chrono::Utc::now().to_rfc3339()
    );

    // 设置 git 提交哈希
    if let Ok(output) = std::process::Command::new("git")
        .args(["rev-parse", "HEAD"])
        .output()
    {
        if output.status.success() {
            let hash = String::from_utf8_lossy(&output.stdout);
            println!("cargo:rustc-env=GIT_COMMIT={}", hash.trim());
        }
    }

    // 触发重新构建
    println!("cargo:rerun-if-changed=.git/HEAD");
}
```

## 测试组织

```rust
// shared/shared-testing/src/lib.rs
pub mod fixtures;
pub mod mocks;
pub mod helpers;

pub use fixtures::*;
pub use mocks::*;
pub use helpers::*;
```

```rust
// shared/shared-testing/src/fixtures.rs
use shared_types::UserId;
use user_core::User;
use chrono::Utc;

pub fn create_test_user() -> User {
    User {
        id: UserId::new(),
        email: "test@example.com".to_string(),
        username: "testuser".to_string(),
        display_name: Some("Test User".to_string()),
        created_at: Utc::now(),
        updated_at: Utc::now(),
    }
}

pub fn create_test_user_with_email(email: &str) -> User {
    let mut user = create_test_user();
    user.email = email.to_string();
    user
}
```

## 检查清单

```markdown
### 工作空间组织检查清单
- [ ] 使用 resolver = "2"
- [ ] 所有依赖在 [workspace.dependencies] 中集中管理
- [ ] 子 crate 使用 { workspace = true } 引用依赖
- [ ] 按业务域（子系统）组织，而非技术层
- [ ] 每个子系统有 core/api/worker 分离
- [ ] 共享类型在 shared-types crate 中
- [ ] 共享基础设施（配置、可观测性）在 shared/ 中
- [ ] 子系统间通过定义的接口通信
- [ ] 测试工具在 shared-testing crate 中
- [ ] 使用 workspace.lints 统一代码检查规则
- [ ] 发布配置包含 LTO 和优化
```
