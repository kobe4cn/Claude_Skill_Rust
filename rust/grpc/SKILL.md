---
description: >-
  Use when building gRPC services in Rust. Provides Prost/Tonic 0.13+ patterns
  including Inner types, MessageSanitizer trait, code generation, and service implementation.
  Triggers: gRPC, protobuf, tonic, prost, proto files.
---

# Rust gRPC/Protobuf Patterns

## Quick Reference

This skill provides guidance for gRPC services with Prost/Tonic:
- Prost/Tonic 0.13+ configuration
- Inner data structures pattern
- MessageSanitizer trait
- Clean service implementations
- gRPC reflection and health checks

## Key Requirements

### Prost/Tonic Standards
- Use prost/tonic 0.13+ versions
- Clean code generation with organized pb module
- Inner data structures for business logic
- MessageSanitizer trait for data transformation

## Technology Stack

```toml
[dependencies]
prost = "0.13"
prost-types = "0.13"
tonic = { version = "0.13", features = ["gzip", "tls"] }

[build-dependencies]
prost-build = "0.13"
tonic-build = { version = "0.13", features = ["prost"] }

[dev-dependencies]
tonic-health = "0.13"
tonic-reflection = "0.13"
```

## Inner Types Pattern

```rust
// Generated protobuf message has many Option<T> fields
// Inner types provide clean, non-optional interfaces

#[derive(Debug, Clone, Default, TypedBuilder)]
pub struct HelloRequestInner {
    #[builder(setter(into))]
    pub name: String,
    #[builder(default = "en".to_string())]
    pub language: String,
    #[builder(default_code = "Utc::now()")]
    pub request_time: DateTime<Utc>,
}
```

## MessageSanitizer Trait

```rust
pub trait MessageSanitizer {
    type Output;
    fn sanitize(self) -> Self::Output;
}

impl MessageSanitizer for HelloRequest {
    type Output = HelloRequestInner;

    fn sanitize(self) -> Self::Output {
        HelloRequestInner::builder()
            .name(self.name)
            .language(self.language.unwrap_or_default())
            .request_time(/* convert timestamp */)
            .build()
    }
}

// Use From trait for reverse conversion
impl From<HelloReplyInner> for HelloReply {
    fn from(inner: HelloReplyInner) -> Self {
        Self {
            message: inner.message,
            // ... convert fields back
        }
    }
}
```

## Service Implementation Pattern

```rust
// Business logic - clean and testable
impl GreeterService {
    pub async fn say_hello_inner(
        &self,
        request: HelloRequestInner,
    ) -> Result<HelloReplyInner> {
        // Pure business logic using Inner types
        let greeting = format!("Hello, {}!", request.name);
        Ok(HelloReplyInner::builder().message(greeting).build())
    }
}

// gRPC trait - thin wrapper
#[tonic::async_trait]
impl GreeterServiceTrait for GreeterService {
    async fn say_hello(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<HelloReply>, Status> {
        let inner = request.into_inner().sanitize();
        match self.say_hello_inner(inner).await {
            Ok(reply) => Ok(Response::new(reply.into())),
            Err(e) => Err(Status::internal(e.to_string())),
        }
    }
}
```

## build.rs Setup

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pb_dir = PathBuf::from("src/pb");

    tonic_build::configure()
        .out_dir(&pb_dir)
        .format(true)
        .build_server(true)
        .build_client(true)
        .extern_path(".google.protobuf.Timestamp", "::prost_types::Timestamp")
        .file_descriptor_set_path(&pb_dir.join("descriptor.bin"))
        .compile(&["proto/service.proto"], &["proto"])?;

    Ok(())
}
```

## Anti-Patterns to Avoid

- Using Option<T> throughout business logic
- Mixing protobuf types with business logic
- Not implementing MessageSanitizer
- Missing gRPC reflection for discovery
- Blocking operations in async handlers

## See Also

- `reference.md` - Complete patterns, testing, and server setup
