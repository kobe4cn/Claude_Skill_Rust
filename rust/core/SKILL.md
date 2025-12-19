---
description: >-
  Use when working with any Rust project. Provides core Rust 2024 development standards
  including code quality, dependencies, type system, performance, security, API design,
  and design patterns. Triggers: Rust, Cargo.toml, *.rs files, new Rust project.
---

# Rust Core Development Standards

## Quick Reference

This skill provides foundational Rust development guidance covering:
- Code quality and organization
- Dependency management
- Type system best practices
- Performance optimization
- Security patterns
- API design principles
- Design patterns

## Key Principles

### Rust 2024 Edition
- Always use `edition = "2024"` in Cargo.toml
- Never use `unsafe` code without explicit justification
- No `unwrap()` or `expect()` in production code

### Code Organization
- **Functionality-based files**: Use `user.rs`, `product.rs` instead of `models.rs`, `types.rs`
- **File size limits**: Maximum 500 lines per file (excluding tests)
- **Function size**: Maximum 150 lines per function
- **Single Responsibility**: Each module should have one clear purpose

### Serde Configuration
```rust
// Always use camelCase for JSON serialization
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ApiResponse {
    pub user_id: String,
    pub created_at: DateTime<Utc>,
}
```

### Naming Conventions
```rust
// Good: Clear, specific names
pub struct UserService;
pub struct ProductCatalog;

// Bad: Avoid these patterns
pub struct UserServiceImpl;  // No "Impl" suffix
pub struct Helper;           // Too generic
```

## Technology Stack

### Error Handling
- **Libraries**: Use `thiserror` for structured error types
- **Binaries**: Use `anyhow` for rich context

### Core Dependencies
```toml
[dependencies]
anyhow = "1.0"
thiserror = "2.0"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.45", features = ["macros", "rt-multi-thread"] }
```

### Build Verification
Always run in order:
```bash
cargo build
cargo test
cargo clippy
```

## Anti-Patterns to Avoid

- Generic file names (`models.rs`, `utils.rs`, `helpers.rs`)
- Implementation suffixes (`UserValidatorImpl`)
- `unwrap()` or `expect()` in production
- Files exceeding 500 lines
- Mixing concerns in single files
- Wildcard imports (`use serde::*`)

## Type System Essentials

### Newtype Pattern
```rust
// Strong typing for domain concepts
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(uuid::Uuid);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ProductId(uuid::Uuid);

// Compiler prevents mixing up IDs
fn process_order(user_id: UserId, product_id: ProductId) -> OrderId
```

### Phantom Types for State
```rust
pub struct Draft;
pub struct Published;

pub struct Document<State> {
    id: DocumentId,
    content: String,
    _state: PhantomData<State>,
}

impl Document<Draft> {
    pub fn publish(self) -> Document<Published> { /* ... */ }
}
```

## See Also

- `reference.md` - Complete patterns, checklists, and detailed examples
