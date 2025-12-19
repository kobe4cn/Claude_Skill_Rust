---
description: >-
  (project) Rust 开发技能集合。提供生产级 Rust 2024 开发标准和最佳实践指导，
  包括核心标准、Web 框架、数据库、CLI、gRPC、并发、配置、可观测性、错误处理
  和工作空间组织。当用户处理任何 Rust 项目时自动激活。
  触发条件：Rust, Cargo.toml, *.rs, cargo build, cargo test。
---

# Rust 开发技能

这是一个完整的 Rust 生产级开发技能集合，基于真实生产经验构建，提供从项目创建到部署的全面指导。

## 技能概览

本技能包含 18 个专注领域的子技能：

| 子技能 | 用途 | 触发关键词 |
|--------|------|-----------|
| **core** | 核心 Rust 2024 标准 | 代码质量、类型系统、性能、安全 |
| **api-design** | API 设计模式 | API 设计, 函数签名, Builder 模式, Trait 设计 |
| **design-patterns** | 设计模式 | 策略, 命令, 观察者, Actor, 工厂模式 |
| **axum** | Axum 0.8+ Web 框架 | Web, HTTP, REST, OpenAPI |
| **database** | SQLx 数据库模式 | SQLx, PostgreSQL, 仓储, 迁移 |
| **cli** | Clap 4.0+ 命令行 | CLI, 命令行, 参数解析 |
| **grpc** | Prost/Tonic gRPC | gRPC, Protobuf, Tonic |
| **concurrency** | Tokio 异步并发 | 并发, 异步, DashMap, Channel |
| **config** | figment 配置管理 | 配置, 热重载, YAML, TOML |
| **observability** | 可观测性 | 指标, 追踪, Prometheus, 健康检查 |
| **errors** | 错误处理 | thiserror, anyhow, Result |
| **workspace** | 多 crate 工作空间 | 工作空间, monorepo, 子系统 |
| **utilities** | 工具库 | JWT, 认证, 随机生成, derive_more |
| **tools-config** | 工具与配置 | tracing, 日志, MiniJinja, 模板 |
| **http-client** | HTTP 客户端 | reqwest, REST 客户端, API 集成 |
| **testing** | 测试标准 | 单元测试, 集成测试, proptest, 基准测试 |
| **simple-crate** | 单 crate 项目 | 项目结构, minimal main, 二进制/库 crate |
| **lib-template** | 库项目模板 | cargo generate, lib template, 新库项目 |

## 核心技术栈

### 首选技术

| 领域 | 推荐 | 版本 |
|------|------|------|
| **Web 框架** | Axum | 0.8+ |
| **数据库** | SQLx | 0.8+ |
| **CLI** | Clap | 4.0+ |
| **gRPC** | Tonic/Prost | 0.13+ |
| **并发** | Tokio + DashMap | 1.45+ / 6.0+ |
| **配置** | figment + arc-swap | 0.10+ / 1.0+ |
| **可观测性** | Prometheus + OpenTelemetry | - |
| **错误处理** | thiserror (库) / anyhow (二进制) | 2.0+ / 1.0+ |

### 禁止使用

- `rusqlite` 或 `tokio-postgres`（使用 SQLx）
- `Mutex<HashMap>`（使用 DashMap）
- `unwrap()` 或 `expect()` 在生产代码中
- `unsafe` 代码（除非绝对必要并有文档说明）

## 核心原则

### 1. Rust 2024 版本
```toml
[package]
edition = "2024"
```

### 2. 代码组织
- **基于功能的文件**：`user.rs`, `product.rs` 而非 `models.rs`, `types.rs`
- **文件限制**：最大 500 行（不含测试）
- **函数限制**：最大 150 行
- **单一职责**：每个模块一个明确目的

### 3. Serde 配置
```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ApiResponse {
    pub user_id: String,
    pub created_at: DateTime<Utc>,
}
```

### 4. 构建验证
```bash
cargo build
cargo test
cargo clippy -- -D warnings
```

## 规则加载层级

- `rust/SKILL.md` 作为总体入口
- `core` / `errors` / `testing` 始终相关
- 根据复杂度选择 `simple-crate` 或 `workspace`
- 根据需求加载特性技能（axum/database/grpc/...）

## 项目复杂度判定

### 简单项目 (Single Crate)
- **单一功能域**：CLI 工具、库、简单服务
- **代码规模**：预计 < 10,000 行
- **团队规模**：1-3 人
- **依赖集成**：较少
- **部署形态**：单个二进制或库
- **加载**：`core` + `errors` + `testing` + `simple-crate` + 按需特性技能

### 复杂项目 (Workspace)
- **多个功能域**：认证、业务、数据处理等
- **代码规模**：预计 > 10,000 行
- **团队规模**：4+ 人或多团队
- **共享组件**：需要共享库或多可部署单元
- **部署形态**：多服务/多二进制
- **加载**：`core` + `errors` + `testing` + `workspace` + 按需特性技能

## 特性检测清单

### Web Framework Requirements
- [ ] HTTP 服务或 REST API
- [ ] OpenAPI/Swagger 文档
- [ ] SSE/WebSocket
- [ ] → 加载 `axum`

### Database Requirements
- [ ] SQL 数据库访问
- [ ] 迁移/连接池/仓储模式
- [ ] → 加载 `database`

### Concurrency Requirements
- [ ] 异步任务、并发处理
- [ ] 共享状态或锁竞争
- [ ] → 加载 `concurrency`

### Configuration Management Requirements
- [ ] 多格式配置 (YAML/TOML/JSON)
- [ ] 热重载与配置校验
- [ ] → 加载 `config`

### Observability Requirements
- [ ] 指标、追踪、健康检查
- [ ] → 加载 `observability`

### Tools & Config Requirements
- [ ] 结构化日志或 tracing
- [ ] 模板渲染、JSONPath 提取
- [ ] → 加载 `tools-config`

### Serialization Requirements
- [ ] JSON 序列化或外部 API
- [ ] camelCase 规则
- [ ] → 使用 `core` 的 Serde 规范

### Builder Pattern Requirements
- [ ] 复杂结构体 (4+ 字段)
- [ ] 可选字段与流式构建
- [ ] → 使用 `utilities` / `api-design` 的 TypedBuilder 模式

### Utility Libraries Requirements
- [ ] JWT/认证
- [ ] 随机数据生成、派生宏
- [ ] → 加载 `utilities`

### CLI Application Requirements
- [ ] 多子命令 CLI
- [ ] 交互式提示或进度条
- [ ] → 加载 `cli`

### Protobuf & gRPC Requirements
- [ ] Protobuf 数据结构
- [ ] gRPC 服务与客户端
- [ ] → 加载 `grpc`

### HTTP Client Requirements
- [ ] 外部 API 集成
- [ ] 认证头与重试策略
- [ ] → 加载 `http-client`

## 规则加载示例

- Web API：`core` + `errors` + `testing` + `axum` + `database` + `utilities`
- CLI 工具：`core` + `errors` + `testing` + `cli` + `utilities`
- gRPC 服务：`core` + `errors` + `testing` + `grpc` + `observability`
- 配置管理：`core` + `errors` + `testing` + `config` + `observability`
- 外部 API 客户端：`core` + `errors` + `testing` + `http-client`

## 规则模块列表

| 模块 | 文件 | 说明 |
|------|------|------|
| **Core** | `core/code-quality.mdc` | Rust 2024 与代码质量规范 |
| **Core** | `core/dependencies.mdc` | 依赖管理与 workspace 规范 |
| **Core** | `core/type-system.mdc` | 类型系统模式（newtype/phantom） |
| **Core** | `core/performance.mdc` | 性能优化与内存管理 |
| **Core** | `core/security.mdc` | 安全模式（加密/哈希/密钥） |
| **Core** | `core/api-design.mdc` | API 设计与 builder 模式 |
| **Core** | `core/design-patterns.mdc` | 设计模式与 actor 模型 |
| **Simple** | `simple/single-crate.mdc` | 单 crate 项目结构 |
| **Complex** | `complex/workspace.mdc` | 多 crate workspace 管理 |
| **Web** | `features/axum.mdc` | Axum 0.8 + OpenAPI |
| **Database** | `features/database.mdc` | SQLx 仓储与测试 |
| **CLI** | `features/cli.mdc` | Clap 4.0 + 子命令 |
| **Protobuf & gRPC** | `features/protobuf-grpc.mdc` | Prost/Tonic 0.13+ |
| **Concurrency** | `features/concurrency.mdc` | Tokio 与并发模式 |
| **Configuration** | `features/configuration.mdc` | 多格式配置与热重载 |
| **Observability** | `features/observability.mdc` | 指标/追踪/健康检查 |
| **Tools & Config** | `features/tools-and-config.mdc` | tracing/YAML/MiniJinja |
| **Utilities** | `features/utilities.mdc` | JWT/derive/工具库 |
| **HTTP Client** | `features/http-client.mdc` | reqwest/重试/错误处理 |
| **Testing** | `quality/testing.mdc` | 单元/集成测试标准 |
| **Errors** | `quality/error-handling.mdc` | thiserror/anyhow 错误处理 |

## 开发前验证清单

```markdown
✓ RUST PROJECT VERIFICATION
- 项目复杂度已判定？[SIMPLE/COMPLEX]
- 特性已识别？[列出]
- 规则已加载？[YES/NO]
- Cargo.toml 结构已确认？[YES/NO]
- 错误处理策略已选择？[thiserror/anyhow]
- 使用 Rust 2024 版本？[YES/NO]
- 无 unsafe 计划？[YES/NO]
- Workspace 依赖已配置？[YES/NO]
```

## 快速开始

### 新 Web API 项目
```bash
cargo new my_api --bin
cd my_api
cargo add axum tokio serde sqlx anyhow
```

参考子技能：`core`, `errors`, `testing`, `axum`, `database`

### 新 CLI 工具
```bash
cargo new my_cli --bin
cd my_cli
cargo add clap anyhow colored
```

参考子技能：`core`, `errors`, `testing`, `cli`

### 新 gRPC 服务
```bash
cargo new my_service --bin
cd my_service
cargo add tonic prost tokio
```

参考子技能：`core`, `errors`, `testing`, `grpc`, `observability`

### 新库项目（使用模板）
```bash
# 安装 cargo-generate（如果未安装）
cargo install cargo-generate

# 从模板创建库项目
cargo generate --git https://github.com/kobe4cn/rust-lib-template
```

参考子技能：`core`, `errors`, `testing`, `lib-template`

## 子技能详情

每个子技能包含：
- `SKILL.md` - 快速参考和核心模式
- `reference.md` - 完整实现细节和检查清单

浏览 `./core/`, `./axum/`, `./database/` 等子目录获取详细指导。

## 来源

这些技能从 `.cursor/rules/rust/` 中的 22 个 MDC 规则文件转换而来，整合为 18 个专注领域的技能（包括 `lib-template` 用于标准化库项目初始化）。
