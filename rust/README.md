# Rust 开发技能集合

这是一组用于 Claude Code 的 Rust 开发技能，基于生产级最佳实践构建。

## 目录结构

```
.claude/skills/rust/
├── SKILL.md              # 总体技能描述（自动发现入口）
├── README.md             # 本文件
├── core/                 # 核心 Rust 2024 标准
│   ├── SKILL.md
│   └── reference.md
├── api-design/           # API 设计模式
│   ├── SKILL.md
│   └── reference.md
├── design-patterns/      # 设计模式 (策略/命令/观察者/Actor)
│   ├── SKILL.md
│   └── reference.md
├── axum/                 # Axum 0.8+ Web 框架
│   ├── SKILL.md
│   └── reference.md
├── database/             # SQLx 数据库访问
│   ├── SKILL.md
│   └── reference.md
├── cli/                  # Clap 4.0+ 命令行应用
│   ├── SKILL.md
│   └── reference.md
├── grpc/                 # Prost/Tonic gRPC 服务
│   ├── SKILL.md
│   └── reference.md
├── concurrency/          # Tokio 异步并发
│   ├── SKILL.md
│   └── reference.md
├── config/               # figment 配置管理
│   ├── SKILL.md
│   └── reference.md
├── observability/        # Prometheus/OpenTelemetry
│   ├── SKILL.md
│   └── reference.md
├── errors/               # thiserror/anyhow 错误处理
│   ├── SKILL.md
│   └── reference.md
├── workspace/            # 多 crate 工作空间
│   ├── SKILL.md
│   └── reference.md
├── utilities/            # JWT/随机生成/derive_more
│   ├── SKILL.md
│   └── reference.md
├── tools-config/         # tracing/YAML配置/MiniJinja
│   ├── SKILL.md
│   └── reference.md
├── http-client/          # reqwest HTTP 客户端
│   ├── SKILL.md
│   └── reference.md
├── testing/              # 测试标准 (单元/集成/属性/基准)
│   ├── SKILL.md
│   └── reference.md
└── simple-crate/         # 单 crate 项目结构
    ├── SKILL.md
    └── reference.md
```

## 技能列表

| 技能 | 描述 | 触发关键词 |
|------|------|-----------|
| **rust** (总体) | Rust 开发技能集合 | Rust, Cargo.toml, *.rs |
| **core** | 核心 Rust 2024 开发标准 | 代码质量, 类型系统, 性能 |
| **api-design** | API 设计模式 | API 设计, 函数签名, Builder 模式, Trait 设计 |
| **design-patterns** | 设计模式 | 策略, 命令, 观察者, Actor, 工厂模式 |
| **axum** | Axum 0.8+ Web 框架模式 | Axum, Web, HTTP, REST, OpenAPI |
| **database** | SQLx 数据库访问模式 | SQLx, 数据库, PostgreSQL, 迁移 |
| **cli** | Clap 4.0+ 命令行应用模式 | CLI, Clap, 命令行, 参数解析 |
| **grpc** | Prost/Tonic gRPC 服务模式 | gRPC, Protobuf, Tonic, Prost |
| **concurrency** | Tokio 异步并发模式 | 并发, 异步, Tokio, DashMap, Channel |
| **config** | figment 配置管理模式 | 配置, figment, 热重载, YAML, TOML |
| **observability** | Prometheus/OpenTelemetry 可观测性 | 指标, 追踪, Prometheus, 健康检查 |
| **errors** | thiserror/anyhow 错误处理 | 错误处理, thiserror, anyhow, Result |
| **workspace** | 多 crate 工作空间组织 | 工作空间, 多 crate, monorepo, 子系统 |
| **utilities** | 工具库 | JWT, 认证, 随机生成, derive_more, typed-builder |
| **tools-config** | 工具与配置 | tracing, 日志, MiniJinja, 模板, JSONPath |
| **http-client** | HTTP 客户端 | reqwest, REST 客户端, API 集成, HTTP 请求 |
| **testing** | 测试标准 | 单元测试, 集成测试, proptest, criterion, mockall |
| **simple-crate** | 单 crate 项目结构 | 项目结构, minimal main, 二进制 crate, 库 crate |

## 使用方式

### 自动发现

Claude Code 会根据以下条件自动激活技能：

1. **总体技能**：检测到 Cargo.toml 或 *.rs 文件时激活 `rust/SKILL.md`
2. **子技能**：根据问题关键词激活对应的子技能
3. **基础技能**：`core`、`errors`、`testing` 始终相关，建议默认阅读

### 技能层级

```
rust/SKILL.md (总体入口)
├── core/SKILL.md (始终相关)
├── errors/SKILL.md (始终相关)
├── testing/SKILL.md (始终相关)
└── [特性技能] (按需激活)
```

### 示例场景

| 用户请求 | 激活的技能 |
|----------|-----------|
| "创建新的 Rust 项目" | rust, core, errors, testing |
| "添加 Axum Web API" | rust, core, errors, testing, axum |
| "实现 SQLx 数据库层" | rust, core, errors, testing, database |
| "构建 CLI 工具" | rust, core, errors, testing, cli |
| "设置 gRPC 服务" | rust, core, errors, testing, grpc |
| "添加 Prometheus 指标" | rust, core, errors, testing, observability |

## 技术栈

这些技能基于以下现代 Rust 技术栈：

| 领域 | 推荐技术 | 版本 |
|------|----------|------|
| Web 框架 | Axum | 0.8+ |
| 数据库 | SQLx | 0.8+ |
| CLI | Clap with derive | 4.0+ |
| gRPC | Prost/Tonic | 0.13+ |
| 并发 | Tokio + DashMap | 1.45+ / 6.0+ |
| 配置 | figment + arc-swap | 0.10+ / 1.0+ |
| 可观测性 | Prometheus + OpenTelemetry | - |
| 错误处理 | thiserror (库) / anyhow (二进制) | 2.0+ / 1.0+ |

### 禁止使用

- `rusqlite` 或 `tokio-postgres`（使用 SQLx）
- `Mutex<HashMap>`（使用 DashMap）
- `unwrap()` 或 `expect()` 在生产代码中
- `unsafe` 代码（除非绝对必要）

## 文件说明

每个技能目录包含两个文件：

- **SKILL.md** - 带有 YAML frontmatter 的快速参考
  - 包含触发描述
  - 核心模式和代码示例
  - 技术栈要求
  - 反模式警告

- **reference.md** - 完整参考文档
  - 详细实现模式
  - 完整代码示例
  - 检查清单
  - 测试策略

## 来源与规则模块

这些技能从 `.cursor/rules/rust/` 中的 22 个 MDC 规则文件转换而来：

| 模块 | 规则文件 | 对应技能 | 说明 |
|------|----------|----------|------|
| **Root** | `main.mdc` | rust (总体) | 规则加载体系、复杂度判定、特性检测 |
| **Core** | `core/code-quality.mdc` | core | Rust 2024 与代码质量规范 |
| **Core** | `core/dependencies.mdc` | core | 依赖管理与 workspace 规范 |
| **Core** | `core/type-system.mdc` | core | 类型系统模式（newtype/phantom） |
| **Core** | `core/performance.mdc` | core | 性能优化与内存管理 |
| **Core** | `core/security.mdc` | core | 安全模式（加密/哈希/密钥） |
| **Core** | `core/api-design.mdc` | api-design | API 设计与 builder 模式 |
| **Core** | `core/design-patterns.mdc` | design-patterns | 设计模式与 actor 模型 |
| **Simple** | `simple/single-crate.mdc` | simple-crate | 单 crate 项目结构 |
| **Complex** | `complex/workspace.mdc` | workspace | 多 crate workspace 管理 |
| **Web** | `features/axum.mdc` | axum | Axum 0.8 + OpenAPI |
| **Database** | `features/database.mdc` | database | SQLx 仓储与测试 |
| **CLI** | `features/cli.mdc` | cli | Clap 4.0 + 子命令 |
| **Protobuf & gRPC** | `features/protobuf-grpc.mdc` | grpc | Prost/Tonic 0.13+ |
| **Concurrency** | `features/concurrency.mdc` | concurrency | Tokio 与并发模式 |
| **Configuration** | `features/configuration.mdc` | config | 多格式配置与热重载 |
| **Observability** | `features/observability.mdc` | observability | 指标/追踪/健康检查 |
| **Tools & Config** | `features/tools-and-config.mdc` | tools-config | tracing/YAML/MiniJinja |
| **Utilities** | `features/utilities.mdc` | utilities | JWT/derive/工具库 |
| **HTTP Client** | `features/http-client.mdc` | http-client | reqwest/重试/错误处理 |
| **Testing** | `quality/testing.mdc` | testing | 单元/集成测试标准 |
| **Errors** | `quality/error-handling.mdc` | errors | thiserror/anyhow 错误处理 |

## 同步维护

Cursor 规则 (`.cursor/rules/rust/`) 是真实来源。当规则更新时：

1. 更新对应的 `SKILL.md` 和 `reference.md`
2. 保持技术栈版本同步
3. 验证示例代码可编译
