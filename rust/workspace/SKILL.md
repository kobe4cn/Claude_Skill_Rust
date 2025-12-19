---
description: >-
  Use when organizing multi-crate Rust workspaces. Provides subsystem-based architecture,
  dependency management, and domain-driven design patterns.
  Triggers: workspace, multi-crate, Cargo.toml workspace, monorepo, subsystems.
---

# Rust Workspace Organization

## Quick Reference

This skill provides guidance for complex Rust projects:
- Subsystem-based architecture
- Workspace dependency management
- Domain-driven crate organization
- Cross-subsystem communication patterns

## Key Principles

### Workspace Architecture
- Organize by business domain (subsystems), not by technical layer
- Each subsystem has: core (logic), api (HTTP), worker (background)
- Shared infrastructure for cross-cutting concerns
- No direct database access between subsystems

## Directory Structure

```
service-platform/
├── Cargo.toml                 # Workspace config
├── shared/                    # Cross-cutting infrastructure
│   ├── shared-types/          # Domain primitives
│   ├── shared-config/         # Configuration
│   └── shared-observability/  # Metrics & tracing
├── services/                  # Business services
│   ├── user-management/
│   │   ├── user-core/         # Business logic
│   │   ├── user-api/          # HTTP handlers
│   │   └── user-worker/       # Background jobs
│   └── order-management/
│       ├── order-core/
│       ├── order-api/
│       └── order-processor/
└── platform/                  # Infrastructure
    ├── api-gateway/
    └── admin-cli/
```

## Workspace Cargo.toml

```toml
[workspace]
resolver = "2"
members = [
    "shared/shared-types",
    "shared/shared-config",
    "services/user-management/user-core",
    "services/user-management/user-api",
    "platform/api-gateway",
]

[workspace.dependencies]
tokio = { version = "1.45", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
sqlx = { version = "0.8", features = ["postgres"] }

[workspace.package]
edition = "2024"
version = "0.1.0"
```

## Crate Dependencies

```toml
# services/user-management/user-api/Cargo.toml
[package]
name = "user-api"
version.workspace = true
edition.workspace = true

[dependencies]
user-core = { path = "../user-core" }
shared-types = { path = "../../../shared/shared-types" }
tokio = { workspace = true }
axum = { workspace = true }
```

## Shared Types Pattern

```rust
// shared/shared-types/src/ids.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct UserId(pub Uuid);

impl UserId {
    pub fn new() -> Self {
        Self(Uuid::new_v4())
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct OrderId(pub Uuid);
```

## Service Communication

```rust
use tokio::sync::broadcast;

pub struct ServiceManager<T> {
    config: Arc<ArcSwap<T>>,
    components: Vec<Arc<dyn ServiceComponent>>,
}

impl<T> ServiceManager<T> {
    pub fn update_config(&self, new_config: T) {
        self.config.store(Arc::new(new_config));
        for component in &self.components {
            component.on_config_update();
        }
    }
}

pub trait ServiceComponent: Send + Sync {
    fn on_config_update(&self);
    fn health_check(&self) -> ServiceHealth;
}
```

## Anti-Patterns to Avoid

- Type-based organization (models/, services/, handlers/)
- Direct cross-subsystem database access
- Circular dependencies between subsystems
- Shared mutable state without proper synchronization
- Missing workspace dependency centralization

## See Also

- `reference.md` - Complete patterns, build scripts, and testing
