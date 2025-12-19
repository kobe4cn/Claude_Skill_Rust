# Rust gRPC/Protobuf 完整参考

## Proto 文件组织

```
proto/
├── common/
│   ├── types.proto         # 共享类型
│   └── errors.proto        # 错误定义
├── services/
│   ├── user.proto          # 用户服务
│   └── order.proto         # 订单服务
└── buf.yaml                # Buf 配置
```

## 完整 build.rs 配置

```rust
use std::path::PathBuf;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let out_dir = PathBuf::from("src/pb");

    // 确保输出目录存在
    std::fs::create_dir_all(&out_dir)?;

    tonic_build::configure()
        .out_dir(&out_dir)
        .format(true)
        .build_server(true)
        .build_client(true)
        // 使用 prost-types 的 Timestamp
        .extern_path(".google.protobuf.Timestamp", "::prost_types::Timestamp")
        .extern_path(".google.protobuf.Duration", "::prost_types::Duration")
        // 生成文件描述符用于反射
        .file_descriptor_set_path(out_dir.join("descriptor.bin"))
        // 类型属性
        .type_attribute(".", "#[derive(serde::Serialize, serde::Deserialize)]")
        .compile_protos(
            &[
                "proto/services/user.proto",
                "proto/services/order.proto",
            ],
            &["proto"],
        )?;

    // 触发重新编译
    println!("cargo:rerun-if-changed=proto/");

    Ok(())
}
```

## Inner 类型完整实现

```rust
use chrono::{DateTime, Utc};
use typed_builder::TypedBuilder;
use uuid::Uuid;

// 生成的 protobuf 消息
pub mod pb {
    include!("pb/user.rs");
}

// 内部业务类型 - 干净且类型安全
#[derive(Debug, Clone, TypedBuilder)]
pub struct CreateUserRequestInner {
    #[builder(setter(into))]
    pub username: String,

    #[builder(setter(into))]
    pub email: String,

    #[builder(default, setter(strip_option))]
    pub display_name: Option<String>,

    #[builder(default_code = "Utc::now()")]
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, TypedBuilder)]
pub struct UserResponseInner {
    pub id: Uuid,

    #[builder(setter(into))]
    pub username: String,

    #[builder(setter(into))]
    pub email: String,

    #[builder(default, setter(strip_option))]
    pub display_name: Option<String>,

    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}
```

## MessageSanitizer 完整 Trait

```rust
use anyhow::{Result, Context};

/// 将 protobuf 消息转换为内部类型的 trait
pub trait MessageSanitizer {
    type Output;
    type Error;

    /// 验证并转换消息
    fn sanitize(self) -> Result<Self::Output, Self::Error>;
}

impl MessageSanitizer for pb::CreateUserRequest {
    type Output = CreateUserRequestInner;
    type Error = anyhow::Error;

    fn sanitize(self) -> Result<Self::Output> {
        // 验证必填字段
        if self.username.is_empty() {
            anyhow::bail!("username is required");
        }
        if self.email.is_empty() {
            anyhow::bail!("email is required");
        }

        // 转换为内部类型
        Ok(CreateUserRequestInner::builder()
            .username(self.username)
            .email(self.email)
            .display_name(if self.display_name.is_empty() {
                None
            } else {
                Some(self.display_name)
            })
            .build())
    }
}

/// 从内部类型转换回 protobuf 消息
impl From<UserResponseInner> for pb::UserResponse {
    fn from(inner: UserResponseInner) -> Self {
        Self {
            id: inner.id.to_string(),
            username: inner.username,
            email: inner.email,
            display_name: inner.display_name.unwrap_or_default(),
            created_at: Some(datetime_to_timestamp(inner.created_at)),
            updated_at: Some(datetime_to_timestamp(inner.updated_at)),
        }
    }
}

fn datetime_to_timestamp(dt: DateTime<Utc>) -> prost_types::Timestamp {
    prost_types::Timestamp {
        seconds: dt.timestamp(),
        nanos: dt.timestamp_subsec_nanos() as i32,
    }
}

fn timestamp_to_datetime(ts: prost_types::Timestamp) -> DateTime<Utc> {
    DateTime::from_timestamp(ts.seconds, ts.nanos as u32)
        .unwrap_or_else(Utc::now)
}
```

## 完整服务实现

```rust
use tonic::{Request, Response, Status};
use std::sync::Arc;

pub struct UserService {
    db: Arc<DatabasePool>,
    metrics: Arc<MetricsCollector>,
}

impl UserService {
    pub fn new(db: Arc<DatabasePool>, metrics: Arc<MetricsCollector>) -> Self {
        Self { db, metrics }
    }

    /// 内部业务逻辑 - 可测试，类型安全
    pub async fn create_user_inner(
        &self,
        request: CreateUserRequestInner,
    ) -> Result<UserResponseInner> {
        // 纯业务逻辑
        let user = self.db.users()
            .create(&request.username, &request.email, request.display_name.as_deref())
            .await
            .context("failed to create user")?;

        self.metrics.increment("users_created");

        Ok(UserResponseInner::builder()
            .id(user.id)
            .username(user.username)
            .email(user.email)
            .display_name(user.display_name)
            .created_at(user.created_at)
            .updated_at(user.updated_at)
            .build())
    }
}

#[tonic::async_trait]
impl pb::user_service_server::UserService for UserService {
    async fn create_user(
        &self,
        request: Request<pb::CreateUserRequest>,
    ) -> Result<Response<pb::UserResponse>, Status> {
        // 提取和清理请求
        let inner = request.into_inner()
            .sanitize()
            .map_err(|e| Status::invalid_argument(e.to_string()))?;

        // 调用业务逻辑
        match self.create_user_inner(inner).await {
            Ok(response) => Ok(Response::new(response.into())),
            Err(e) => {
                tracing::error!("create_user failed: {:?}", e);
                Err(Status::internal("internal error"))
            }
        }
    }
}
```

## 服务器设置与健康检查

```rust
use tonic::transport::Server;
use tonic_health::server::health_reporter;
use tonic_reflection::server::Builder as ReflectionBuilder;

pub async fn run_server(config: &ServerConfig) -> Result<()> {
    let addr = config.listen_addr.parse()?;

    // 健康检查
    let (mut health_reporter, health_service) = health_reporter();
    health_reporter.set_serving::<pb::user_service_server::UserServiceServer<UserService>>().await;

    // 反射服务（用于 grpcurl 等工具）
    let reflection_service = ReflectionBuilder::configure()
        .register_encoded_file_descriptor_set(pb::FILE_DESCRIPTOR_SET)
        .build_v1()?;

    // 用户服务
    let user_service = UserService::new(db.clone(), metrics.clone());

    tracing::info!("gRPC server listening on {}", addr);

    Server::builder()
        .add_service(health_service)
        .add_service(reflection_service)
        .add_service(pb::user_service_server::UserServiceServer::new(user_service))
        .serve_with_shutdown(addr, shutdown_signal())
        .await?;

    Ok(())
}

async fn shutdown_signal() {
    tokio::signal::ctrl_c()
        .await
        .expect("failed to listen for shutdown signal");
    tracing::info!("shutdown signal received");
}
```

## 流式 RPC 模式

```rust
use tokio_stream::wrappers::ReceiverStream;
use tokio::sync::mpsc;

#[tonic::async_trait]
impl pb::stream_service_server::StreamService for StreamServiceImpl {
    type ListUsersStream = ReceiverStream<Result<pb::UserResponse, Status>>;

    async fn list_users(
        &self,
        request: Request<pb::ListUsersRequest>,
    ) -> Result<Response<Self::ListUsersStream>, Status> {
        let (tx, rx) = mpsc::channel(128);

        let db = self.db.clone();
        tokio::spawn(async move {
            let mut cursor = db.users().stream_all().await;

            while let Some(user) = cursor.next().await {
                match user {
                    Ok(u) => {
                        let response = UserResponseInner::from(u).into();
                        if tx.send(Ok(response)).await.is_err() {
                            break; // 客户端已断开
                        }
                    }
                    Err(e) => {
                        let _ = tx.send(Err(Status::internal(e.to_string()))).await;
                        break;
                    }
                }
            }
        });

        Ok(Response::new(ReceiverStream::new(rx)))
    }
}
```

## 客户端使用

```rust
use tonic::transport::Channel;

pub struct UserClient {
    inner: pb::user_service_client::UserServiceClient<Channel>,
}

impl UserClient {
    pub async fn connect(addr: &str) -> Result<Self> {
        let channel = Channel::from_shared(addr.to_string())?
            .connect()
            .await?;

        Ok(Self {
            inner: pb::user_service_client::UserServiceClient::new(channel),
        })
    }

    pub async fn create_user(&mut self, request: CreateUserRequestInner) -> Result<UserResponseInner> {
        let pb_request = pb::CreateUserRequest {
            username: request.username,
            email: request.email,
            display_name: request.display_name.unwrap_or_default(),
        };

        let response = self.inner.create_user(pb_request).await?;
        response.into_inner().sanitize()
    }
}
```

## 测试模式

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_create_user_inner() {
        let db = create_test_db().await;
        let metrics = Arc::new(MetricsCollector::new());
        let service = UserService::new(db, metrics);

        let request = CreateUserRequestInner::builder()
            .username("testuser")
            .email("test@example.com")
            .build();

        let response = service.create_user_inner(request).await.unwrap();

        assert_eq!(response.username, "testuser");
        assert_eq!(response.email, "test@example.com");
    }

    #[tokio::test]
    async fn test_message_sanitizer() {
        let pb_request = pb::CreateUserRequest {
            username: "testuser".to_string(),
            email: "test@example.com".to_string(),
            display_name: "Test User".to_string(),
        };

        let inner = pb_request.sanitize().unwrap();

        assert_eq!(inner.username, "testuser");
        assert_eq!(inner.display_name, Some("Test User".to_string()));
    }

    #[tokio::test]
    async fn test_sanitizer_validation() {
        let pb_request = pb::CreateUserRequest {
            username: "".to_string(), // 无效
            email: "test@example.com".to_string(),
            display_name: String::new(),
        };

        let result = pb_request.sanitize();
        assert!(result.is_err());
    }
}
```

## 检查清单

```markdown
### gRPC 服务检查清单
- [ ] build.rs 正确配置 tonic-build
- [ ] 所有消息类型有对应的 Inner 类型
- [ ] MessageSanitizer trait 实现完整
- [ ] From trait 实现用于响应转换
- [ ] 健康检查服务已启用
- [ ] 反射服务已启用（开发/测试环境）
- [ ] 流式 RPC 正确处理背压
- [ ] 错误正确映射到 gRPC Status
- [ ] 业务逻辑与 gRPC 层分离
- [ ] 单元测试覆盖 Inner 方法
```
