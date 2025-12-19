---
description: >-
  Use when implementing design patterns in Rust. Covers creational patterns
  (type-state builder, factory), behavioral patterns (strategy, command, observer),
  structural patterns (adapter, decorator), and concurrency patterns (actor).
  Triggers: design pattern, strategy, command, observer, actor, factory, adapter, decorator.
---

# Rust Design Patterns

## Quick Reference

Essential design patterns for Rust applications, focusing on idiomatic solutions that leverage Rust's ownership system and zero-cost abstractions.

## Creational Patterns

### Type-State Builder
```rust
use std::marker::PhantomData;

pub struct Missing;
pub struct Present;

pub struct ConfigBuilder<HasHost, HasPort> {
    host: Option<String>,
    port: Option<u16>,
    _marker: PhantomData<(HasHost, HasPort)>,
}

impl ConfigBuilder<Missing, Missing> {
    pub fn new() -> Self {
        Self { host: None, port: None, _marker: PhantomData }
    }
}

impl<HasPort> ConfigBuilder<Missing, HasPort> {
    pub fn host(self, host: impl Into<String>) -> ConfigBuilder<Present, HasPort> {
        ConfigBuilder {
            host: Some(host.into()),
            port: self.port,
            _marker: PhantomData,
        }
    }
}

// Only buildable when all required fields present
impl ConfigBuilder<Present, Present> {
    pub fn build(self) -> Config {
        Config { host: self.host.unwrap(), port: self.port.unwrap() }
    }
}
```

### Factory Pattern
```rust
pub trait ConnectionFactory {
    type Connection;
    type Config;
    type Error;

    fn create_connection(config: Self::Config) -> Result<Self::Connection, Self::Error>;
}
```

## Behavioral Patterns

### Strategy Pattern with Enums
```rust
#[derive(Debug, Clone)]
pub enum AuthStrategy {
    Bearer { token: String },
    ApiKey { key: String, header: String },
    Basic { username: String, password: String },
}

impl AuthStrategy {
    pub fn apply_to_request(&self, request: &mut Request) -> Result<(), AuthError> {
        match self {
            AuthStrategy::Bearer { token } => { /* ... */ }
            AuthStrategy::ApiKey { key, header } => { /* ... */ }
            AuthStrategy::Basic { username, password } => { /* ... */ }
        }
        Ok(())
    }
}
```

### Observer Pattern with Async
```rust
use tokio::sync::broadcast;

#[derive(Debug, Clone)]
pub enum DomainEvent {
    UserCreated { user_id: UserId, email: String },
    OrderPlaced { order_id: OrderId, amount: Decimal },
}

pub struct EventBus {
    sender: broadcast::Sender<DomainEvent>,
}

impl EventBus {
    pub fn new() -> Self {
        let (sender, _) = broadcast::channel(1000);
        Self { sender }
    }

    pub async fn publish(&self, event: DomainEvent) {
        let _ = self.sender.send(event);
    }

    pub fn subscribe(&self) -> broadcast::Receiver<DomainEvent> {
        self.sender.subscribe()
    }
}
```

## Concurrency Patterns

### Actor Pattern
```rust
use tokio::sync::{mpsc, oneshot};

pub enum ActorMessage {
    GetUser {
        id: UserId,
        respond_to: oneshot::Sender<Result<User, UserError>>,
    },
}

pub struct UserActor {
    receiver: mpsc::Receiver<ActorMessage>,
    cache: DashMap<UserId, User>,
}

impl UserActor {
    pub async fn run(mut self) {
        while let Some(msg) = self.receiver.recv().await {
            self.handle_message(msg).await;
        }
    }
}

#[derive(Clone)]
pub struct UserActorHandle {
    sender: mpsc::Sender<ActorMessage>,
}
```

## See Also

- `reference.md` - Complete implementations and variations
- `../api-design/` - API design principles
- `../concurrency/` - Tokio patterns
