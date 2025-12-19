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

本技能包含 17 个专注领域的子技能：

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

## 项目类型识别

### 简单项目 (Single Crate)
- 单个 `src/main.rs` 或 `src/lib.rs`
- < 5000 行代码
- 加载：`core`, `errors` + 按需加载特性技能

### 中等复杂度 (Multi-Feature)
- 多个功能模块
- 5000-20000 行代码
- 加载：`core`, `errors` + 多个特性技能

### 复杂项目 (Workspace)
- 多个相关包
- > 20000 行代码
- 加载：全部技能，特别是 `workspace`

## 快速开始

### 新 Web API 项目
```bash
cargo new my_api --bin
cd my_api
cargo add axum tokio serde sqlx anyhow
```

参考子技能：`core`, `axum`, `database`, `errors`

### 新 CLI 工具
```bash
cargo new my_cli --bin
cd my_cli
cargo add clap anyhow colored
```

参考子技能：`core`, `cli`, `errors`

### 新 gRPC 服务
```bash
cargo new my_service --bin
cd my_service
cargo add tonic prost tokio
```

参考子技能：`core`, `grpc`, `errors`, `observability`

## 子技能详情

每个子技能包含：
- `SKILL.md` - 快速参考和核心模式
- `reference.md` - 完整实现细节和检查清单

浏览 `./core/`, `./axum/`, `./database/` 等子目录获取详细指导。

## 来源

这些技能从 `.cursor/rules/rust/` 中的 22 个 MDC 规则文件转换而来，整合为 17 个专注领域的技能。
