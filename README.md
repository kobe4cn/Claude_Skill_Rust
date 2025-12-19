# 📖 Cursor Rust Rules 智能知识库使用手册

## 🎯 概述

本手册指导如何使用基于 Cursor Rust Rules 构建的 Claude Code 智能知识库，在新项目中快速应用生产级 Rust 开发规范。

## 🧩 Skills 安装与使用

本仓库的 Rust Skills 位于 `.claude/skills/rust/`。推荐进行全局安装，确保 Claude Code 在任意项目中都能快速加载。

### 1) 从 GitHub 获取

```bash
# 克隆仓库
git clone https://github.com/kobe4cn/Claude_Skill_Rust.git
cd Claude_Skill_Rust

# 更新（已有仓库时）
git pull
```

如果不使用 Git，可在 GitHub 选择 "Code" → "Download ZIP"，解压后使用同样的目录结构。

### 2) 放置到 Claude Code 技能目录

**全局安装（推荐，适用于所有项目）：**

```
~/.claude/skills/rust/
```

**单项目安装（仅对当前项目生效）：**

```
<project>/.claude/skills/rust/
```

建议两者只保留一个位置，避免重复维护。

### 3) 使用方法

- **自动激活**：当 Claude Code 检测到 `Cargo.toml` 或 `*.rs` 时会自动加载 `rust/SKILL.md`。
- **特性驱动**：描述需求时包含关键字（如 Axum、SQLx、gRPC、testing 等）会触发对应子技能。
- **始终相关**：`core`、`errors`、`testing` 默认适用所有 Rust 项目。

### 3.1) 安装验证

进入任意 Rust 项目目录，确认以下路径存在并包含 `SKILL.md`：

```
~/.claude/skills/rust/SKILL.md
```

然后在 Claude Code 中输入：

```
请使用 rust skills 列出核心技能并说明何时加载。
```

如果返回包含 `core/errors/testing` 的技能说明与加载条件，则安装生效。

### 3.2) 单项目安装验证

进入目标项目目录，确认以下路径存在并包含 `SKILL.md`：

```
<project>/.claude/skills/rust/SKILL.md
```

然后在 Claude Code 中输入：

```
请使用 rust skills 检查本项目是否已加载，并给出当前建议的技能组合。
```

如果返回中包含 `rust` 根技能与匹配的子技能（如 `core/errors/testing`），则单项目安装生效。

### 3.3) 一键安装命令

**全局安装（一行命令）：**

```bash
git clone https://github.com/kobe4cn/Claude_Skill_Rust.git \
  && mkdir -p ~/.claude/skills \
  && cp -R Claude_Skill_Rust/.claude/skills/rust ~/.claude/skills/
```

**单项目安装（一行命令）：**

```bash
git clone https://github.com/kobe4cn/Claude_Skill_Rust.git \
  && mkdir -p .claude/skills \
  && cp -R Claude_Skill_Rust/.claude/skills/rust .claude/skills/
```

如需更新，请在仓库目录执行 `git pull` 后，重复复制命令覆盖旧版本。

### 4) Prompt 模板

```
请使用 Rust skills 帮我完成以下任务：
1) 项目类型与规模：
2) 功能清单：
3) 技术栈偏好：
4) 约束条件（性能/安全/合规）：
```

### 5) Prompt 示例

```
请使用 Rust skills 评估项目复杂度，并给出适合的架构与技能加载建议。
```

```
我要用 Axum + SQLx 做 REST API，请使用 rust skills 生成项目结构和核心代码框架。
```

```
需要 gRPC 服务 + 观测性（metrics + tracing），请按 rust skills 给出实现骨架与依赖配置。
```

```
请使用 rust skills 生成单 crate 的最小可运行模板，并包含测试与错误处理规范。
```

```
请根据 rust skills 检查我的代码是否违反了禁用项（unwrap/unsafe/Mutex<HashMap> 等）。
```

## 🚀 快速开始流程

### 阶段1：需求分析与项目规划 (2-5分钟)

#### 1.1 项目需求收集
当你有一个新的 Rust 项目需求时，首先向 Claude Code 描述：

```
示例对话：
用户："我需要开发一个用户管理系统，包含注册、登录、权限管理功能，需要REST API，预计支持1000+用户"

Claude Code 会自动：
✅ 识别为 Web 服务项目
✅ 评估为中等复杂度
✅ 推荐分层架构 + 数据库集成
```

#### 1.2 自动项目分析
Claude Code 基于 `00-project-detection.md` 进行智能分析：

**项目类型识别：**
- Web 服务（关键词：API、REST、用户系统）
- 规模评估（1000+用户 → 中等复杂度）
- 功能需求（认证、权限 → 安全要求高）

**输出示例：**
```
🔍 项目分析结果：
- 类型：Web 服务
- 复杂度：中等
- 核心功能：用户管理 + REST API + 权限系统
- 推荐架构：分层架构 + JWT认证 + PostgreSQL
- 技术栈：Axum 0.8+ + SQLx + OpenAPI
```

#### 1.3 架构方案推荐
基于 `01-architecture-guide.md`，Claude Code 提供详细架构建议：

```
用户："请提供完整的架构方案"

Claude Code 输出：
🏗️ 推荐架构方案：

技术栈组合：
- Web框架: Axum 0.8 (高性能、类型安全)
- 数据库: SQLx + PostgreSQL (编译时检查)
- 认证: JWT + Argon2 密码哈希
- 配置: figment (多格式支持)
- 文档: utoipa (OpenAPI 3.0)
- 监控: tracing + prometheus

项目结构：
my_user_system/
├── src/
│   ├── handlers/     # API处理器
│   ├── services/     # 业务逻辑
│   ├── models/       # 数据模型
│   ├── repositories/ # 数据访问
│   └── middleware/   # 认证中间件
├── migrations/       # 数据库迁移
└── tests/           # 集成测试
```

### 阶段2：项目初始化 (3-8分钟)

#### 2.1 项目创建和基础配置
```bash
用户："请创建项目脚手架"

Claude Code 执行：
# 1. 创建项目结构
cargo new my_user_system --bin
cd my_user_system

# 2. 基于02-core-standards.md生成 Cargo.toml
```

**自动生成的 Cargo.toml：**
```toml
[package]
name = "my_user_system"
version = "0.1.0"
edition = "2024"  # ✅ Rust 2024标准

[dependencies]
# Web框架 (基于架构推荐)
axum = "0.8"
tokio = { version = "1.0", features = ["full"] }

# 数据库 (遵循SQLx标准，绝不使用rusqlite)
sqlx = { version = "0.8", features = ["runtime-tokio-rustls", "postgres"] }

# 认证和安全
jsonwebtoken = "9.0"
argon2 = "0.5"

# 配置管理
figment = { version = "0.10", features = ["toml", "yaml", "env"] }
serde = { version = "1.0", features = ["derive"] }

# 文档和验证
utoipa = "4.0"
utoipa-swagger-ui = "4.0"
validator = "0.16"

# 错误处理 (二进制项目使用anyhow)
anyhow = "1.0"
thiserror = "1.0"

# 可观测性
tracing = "0.1"
tracing-subscriber = "0.3"
```

#### 2.2 项目结构生成
基于 `03-feature-patterns.md`，自动创建标准项目结构：

```
用户："请生成完整的项目文件结构"

Claude Code 创建：
src/
├── main.rs              # ✅ 最小入口点
├── lib.rs               # ✅ 核心应用逻辑
├── config.rs            # ✅ 配置管理
├── errors.rs            # ✅ 错误定义
├── handlers/
│   ├── mod.rs
│   ├── auth.rs          # 认证处理器
│   └── users.rs         # 用户处理器
├── services/
│   ├── mod.rs
│   ├── auth_service.rs  # 认证业务逻辑
│   └── user_service.rs  # 用户业务逻辑
├── repositories/
│   ├── mod.rs
│   ├── user_repository.rs
│   └── traits.rs        # 仓储接口
├── models/
│   ├── mod.rs
│   ├── user.rs          # 用户模型
│   └── auth.rs          # 认证模型
└── middleware/
    ├── mod.rs
    └── auth.rs          # JWT中间件
```

### 阶段3：核心功能实现 (10-20分钟)

#### 3.1 数据模型实现
```
用户："请实现用户数据模型"

Claude Code 基于02-core-standards.md生成：
```

**src/models/user.rs:**
```rust
// ✅ 遵循Newtype模式
use uuid::Uuid;
use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(pub Uuid);

impl UserId {
    pub fn new() -> Self {
        Self(Uuid::new_v4())
    }
}

#[derive(Debug, Clone, PartialEq)]
pub struct Email(String);

impl Email {
    pub fn new(email: String) -> anyhow::Result<Self> {
        if email.contains('@') && email.len() > 5 {
            Ok(Email(email.to_lowercase()))
        } else {
            anyhow::bail!("Invalid email format")
        }
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

// ✅ 功能导向的模型设计
#[derive(Debug, Clone, sqlx::FromRow)]
pub struct User {
    pub id: UserId,
    pub email: Email,
    pub name: String,
    pub password_hash: String,
    pub role: UserRole,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub updated_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "user_role", rename_all = "lowercase")]
pub enum UserRole {
    Admin,
    User,
    Guest,
}

// ✅ 输入验证结构
#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(email)]
    pub email: String,
    
    #[validate(length(min = 2, max = 50))]
    pub name: String,
    
    #[validate(length(min = 8), custom = "validate_password_strength")]
    pub password: String,
}
```

#### 3.2 仓储层实现
```
用户："请实现用户仓储"

Claude Code 基于03-feature-patterns.md生成：
```

**src/repositories/user_repository.rs:**
```rust
use async_trait::async_trait;
use sqlx::PgPool;
use crate::models::{User, UserId, CreateUserRequest};
use crate::repositories::traits::Repository;

pub struct UserRepository {
    pool: PgPool,
}

impl UserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl Repository<User, UserId> for UserRepository {
    type Error = sqlx::Error;
    
    // ✅ 使用SQLx编译时检查（绝不使用rusqlite）
    async fn create(&self, user: User) -> Result<User, Self::Error> {
        sqlx::query_as!(
            User,
            r#"
            INSERT INTO users (id, email, name, password_hash, role, created_at, updated_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            RETURNING *
            "#,
            user.id.0,
            user.email.as_str(),
            user.name,
            user.password_hash,
            user.role as _,
            user.created_at,
            user.updated_at
        )
        .fetch_one(&self.pool)
        .await
    }
    
    async fn find_by_id(&self, id: UserId) -> Result<Option<User>, Self::Error> {
        sqlx::query_as!(
            User,
            "SELECT * FROM users WHERE id = $1",
            id.0
        )
        .fetch_optional(&self.pool)
        .await
    }
}

// ✅ 业务特定查询方法
impl UserRepository {
    pub async fn find_by_email(&self, email: &str) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as!(
            User,
            "SELECT * FROM users WHERE email = $1",
            email.to_lowercase()
        )
        .fetch_optional(&self.pool)
        .await
    }
}
```

#### 3.3 业务逻辑层实现
```
用户："请实现用户服务层"

Claude Code 基于02-core-standards.md的DRY/SRP原则生成：
```

**src/services/user_service.rs:**
```rust
use std::sync::Arc;
use anyhow::Result;
use crate::models::{User, CreateUserRequest, UserId};
use crate::repositories::UserRepository;
use crate::services::AuthService;

// ✅ 单一职责原则：只负责用户业务逻辑
pub struct UserService {
    repository: Arc<UserRepository>,
    auth_service: Arc<AuthService>,
}

impl UserService {
    pub fn new(
        repository: Arc<UserRepository>,
        auth_service: Arc<AuthService>,
    ) -> Self {
        Self { repository, auth_service }
    }
    
    // ✅ 完整的业务流程，包含验证和错误处理
    pub async fn create_user(&self, request: CreateUserRequest) -> Result<User> {
        // 1. 输入验证
        request.validate()
            .map_err(|e| anyhow::anyhow!("Validation failed: {}", e))?;
            
        // 2. 检查邮箱唯一性
        if let Some(_) = self.repository.find_by_email(&request.email).await? {
            anyhow::bail!("Email already exists");
        }
        
        // 3. 密码哈希（委托给认证服务）
        let password_hash = self.auth_service.hash_password(&request.password).await?;
        
        // 4. 创建用户
        let user = User {
            id: UserId::new(),
            email: Email::new(request.email)?,
            name: request.name,
            password_hash,
            role: UserRole::User,
            created_at: chrono::Utc::now(),
            updated_at: chrono::Utc::now(),
        };
        
        self.repository.create(user).await
            .map_err(|e| anyhow::anyhow!("Failed to create user: {}", e))
    }
}
```

#### 3.4 API 处理器实现
```
用户："请实现REST API处理器"

Claude Code 基于03-feature-patterns.md的Axum模式生成：
```

**src/handlers/users.rs:**
```rust
use axum::{
    extract::{State, Path},
    response::Json,
    http::StatusCode,
};
use utoipa::ToSchema;
use crate::models::{CreateUserRequest, User, UserId};
use crate::AppState;

// ✅ OpenAPI文档集成
#[derive(Serialize, ToSchema)]
pub struct UserResponse {
    pub id: String,
    pub email: String,
    pub name: String,
    pub role: String,
    pub created_at: String,
}

/// Create a new user
#[utoipa::path(
    post,
    path = "/api/v1/users",
    request_body = CreateUserRequest,
    responses(
        (status = 201, description = "User created successfully", body = UserResponse),
        (status = 400, description = "Bad request"),
        (status = 409, description = "Email already exists")
    )
)]
pub async fn create_user(
    State(state): State<AppState>,
    Json(request): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<UserResponse>), StatusCode> {
    
    match state.user_service.create_user(request).await {
        Ok(user) => {
            let response = UserResponse {
                id: user.id.0.to_string(),
                email: user.email.as_str().to_string(),
                name: user.name,
                role: format!("{:?}", user.role),
                created_at: user.created_at.to_rfc3339(),
            };
            Ok((StatusCode::CREATED, Json(response)))
        }
        Err(e) => {
            tracing::error!("Failed to create user: {}", e);
            if e.to_string().contains("already exists") {
                Err(StatusCode::CONFLICT)
            } else {
                Err(StatusCode::BAD_REQUEST)
            }
        }
    }
}
```

### 阶段4：质量保证与测试 (5-10分钟)

#### 4.1 自动化测试生成
```
用户："请生成测试代码"

Claude Code 基于04-quality-assurance.md生成：
```

**tests/integration_test.rs:**
```rust
use axum_test::TestServer;
use sqlx::PgPool;
use testcontainers::{clients, images};
use my_user_system::{create_app, AppConfig};

#[tokio::test]
async fn test_user_registration_flow() {
    // ✅ 使用testcontainers进行隔离测试
    let docker = clients::Cli::default();
    let db_container = docker.run(images::postgres::Postgres::default());
    
    let db_url = format!(
        "postgres://postgres:postgres@127.0.0.1:{}/postgres",
        db_container.get_host_port_ipv4(5432)
    );
    
    let pool = PgPool::connect(&db_url).await.unwrap();
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();
    
    let config = AppConfig::test_config();
    let app = create_app(pool, config).await.unwrap();
    let server = TestServer::new(app).unwrap();
    
    // 测试用户注册
    let create_request = serde_json::json!({
        "email": "test@example.com",
        "name": "Test User",
        "password": "SecurePass123!"
    });
    
    let response = server
        .post("/api/v1/users")
        .json(&create_request)
        .await;
        
    assert_eq!(response.status_code(), 201);
    
    let user: UserResponse = response.json();
    assert_eq!(user.email, "test@example.com");
    assert_eq!(user.name, "Test User");
}
```

#### 4.2 质量检查执行
```bash
用户："请运行质量检查"

Claude Code 执行基于04-quality-assurance.md的检查：

# 1. 代码格式检查
cargo fmt --all -- --check

# 2. Clippy静态分析 (零警告要求)
cargo clippy --all-targets --all-features -- -D warnings

# 3. 单元测试
cargo test

# 4. 安全审计
cargo audit

# 5. 代码覆盖率
cargo tarpaulin --all-features
```

### 阶段5：生产部署准备 (5-10分钟)

#### 5.1 生产配置生成
```
用户："请生成生产部署配置"

Claude Code 基于05-production-checklist.md生成：
```

**Dockerfile:**
```dockerfile
# ✅ 多阶段构建优化
FROM rust:1.75 as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*

# ✅ 安全：非root用户运行
RUN useradd -m -u 1001 appuser
WORKDIR /app
COPY --from=builder /app/target/release/my_user_system /app/
COPY --from=builder /app/config /app/config

RUN chown -R appuser:appuser /app
USER appuser

# ✅ 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
CMD ["./my_user_system"]
```

#### 5.2 生产就绪检查
```bash
用户："检查生产就绪状态"

Claude Code 运行05-production-checklist.md的检查脚本：

✅ Level 1 - 基础就绪:
  ✅ 所有测试通过
  ✅ 代码覆盖率 85% (>80% 要求)
  ✅ 零 Clippy 警告
  ✅ 无安全漏洞
  ✅ 基础监控配置

✅ Level 2 - 运维就绪:
  ✅ 结构化日志 (tracing + JSON)
  ✅ 健康检查端点 (/health)
  ✅ 优雅关闭机制
  ✅ 性能基准通过
  ✅ 负载测试完成

🎉 项目已达到运维级生产就绪标准！
```

## 📈 实际效果示例

### 完整对话流程
```
用户: "我需要开发一个博客系统，支持文章发布、评论、用户管理"

Claude Code: 
🔍 项目分析：Web服务 | 中等复杂度 | 内容管理系统
🏗️ 推荐架构：Axum + PostgreSQL + JWT认证 + 分层架构
⏱️ 预计实现时间：30-45分钟

用户: "开始实施"

Claude Code: 
✅ 项目结构已创建
✅ Cargo.toml配置完成 (Rust 2024标准)
✅ 数据库迁移文件生成
✅ 核心模型实现完成
✅ API处理器生成完成
✅ 认证中间件配置完成
✅ OpenAPI文档自动生成
✅ 集成测试创建完成
✅ Docker配置生成完成

质量检查结果：
✅ 测试覆盖率 88%
✅ 零安全漏洞
✅ 生产就绪度：运维级
```

### 时间对比
- **传统开发方式**：2-3天才能建立基础架构
- **使用知识库**：30-45分钟得到生产就绪的项目

### 质量提升
- **自动应用**最佳实践和设计模式
- **确保遵循**Rust 2024标准
- **内置**安全性和性能优化
- **生成**完整的测试和文档

## 💡 高级使用技巧

### 1. 渐进式复杂度处理
```
用户: "这个项目将来可能要支持微服务架构"

Claude Code: 
基于当前中等复杂度，推荐渐进式架构：
阶段1：单体应用 (当前)
阶段2：模块化单体 (准备拆分)  
阶段3：微服务架构 (根据01-architecture-guide.md)

现在创建时考虑未来拆分，使用DDD边界...
```

### 2. 特定场景优化
```
用户: "这是一个高性能交易系统"

Claude Code:
🚀 检测到高性能需求，应用特殊优化：
- 使用DashMap替代Arc<Mutex<HashMap>>
- 无锁数据结构优先
- 内存池和对象复用
- 批量操作优化
- 性能基准测试强化
```

### 3. 安全要求强化
```
用户: "这个系统处理敏感数据"

Claude Code:
🛡️ 应用企业级安全标准：
- 输入验证强化
- 密钥管理外部化
- 审计日志完整
- 访问控制细粒度
- 安全头部配置
```

这个知识库让 Claude Code 能够像有经验的 Rust 架构师一样工作，确保每个项目都能快速达到生产级质量！
