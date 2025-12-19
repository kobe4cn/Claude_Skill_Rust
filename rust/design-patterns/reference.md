# Rust Design Patterns Reference

## Pattern Selection Strategy

```
Design Challenge?
├─ Object Creation? → Creational Patterns
│   ├─ Complex configuration? → Type-State Builder
│   └─ Multiple implementations? → Factory Pattern
├─ Object Behavior? → Behavioral Patterns
│   ├─ Runtime algorithm selection? → Strategy Pattern
│   ├─ Undo/redo required? → Command Pattern
│   └─ Event-driven architecture? → Observer Pattern
├─ Object Structure? → Structural Patterns
│   ├─ External API integration? → Adapter Pattern
│   └─ Cross-cutting concerns? → Decorator Pattern
└─ Concurrency? → Concurrency Patterns
    └─ Isolated state management? → Actor Pattern
```

## Type-State Builder (Complete)

```rust
use std::marker::PhantomData;

pub struct Missing;
pub struct Present;

pub struct DatabaseConfigBuilder<HasHost, HasPort, HasDatabase> {
    host: Option<String>,
    port: Option<u16>,
    database: Option<String>,
    username: Option<String>,
    _marker: PhantomData<(HasHost, HasPort, HasDatabase)>,
}

impl DatabaseConfigBuilder<Missing, Missing, Missing> {
    pub fn new() -> Self {
        Self {
            host: None, port: None, database: None,
            username: None, _marker: PhantomData,
        }
    }
}

impl<HasPort, HasDatabase> DatabaseConfigBuilder<Missing, HasPort, HasDatabase> {
    pub fn host(self, host: impl Into<String>) -> DatabaseConfigBuilder<Present, HasPort, HasDatabase> {
        DatabaseConfigBuilder {
            host: Some(host.into()), port: self.port,
            database: self.database, username: self.username,
            _marker: PhantomData,
        }
    }
}

impl<HasHost, HasDatabase> DatabaseConfigBuilder<HasHost, Missing, HasDatabase> {
    pub fn port(self, port: u16) -> DatabaseConfigBuilder<HasHost, Present, HasDatabase> {
        DatabaseConfigBuilder {
            host: self.host, port: Some(port),
            database: self.database, username: self.username,
            _marker: PhantomData,
        }
    }
}

impl<HasHost, HasPort> DatabaseConfigBuilder<HasHost, HasPort, Missing> {
    pub fn database(self, db: impl Into<String>) -> DatabaseConfigBuilder<HasHost, HasPort, Present> {
        DatabaseConfigBuilder {
            host: self.host, port: self.port,
            database: Some(db.into()), username: self.username,
            _marker: PhantomData,
        }
    }
}

impl DatabaseConfigBuilder<Present, Present, Present> {
    pub fn build(self) -> DatabaseConfig {
        DatabaseConfig {
            host: self.host.unwrap(),
            port: self.port.unwrap(),
            database: self.database.unwrap(),
            username: self.username,
        }
    }
}
```

## Factory Pattern

```rust
pub trait ConnectionFactory {
    type Connection;
    type Config;
    type Error;

    fn create_connection(config: Self::Config) -> Result<Self::Connection, Self::Error>;
    fn connection_type() -> &'static str;
}

pub struct PostgresFactory;

impl ConnectionFactory for PostgresFactory {
    type Connection = sqlx::PgPool;
    type Config = PostgresConfig;
    type Error = sqlx::Error;

    fn create_connection(config: Self::Config) -> Result<Self::Connection, Self::Error> {
        todo!()
    }

    fn connection_type() -> &'static str { "PostgreSQL" }
}
```

## Strategy Pattern with Enums

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
            AuthStrategy::Bearer { token } => {
                request.headers_mut().insert(
                    "Authorization",
                    format!("Bearer {}", token).parse().unwrap(),
                );
            }
            AuthStrategy::ApiKey { key, header } => {
                request.headers_mut().insert(header.as_str(), key.parse().unwrap());
            }
            AuthStrategy::Basic { username, password } => {
                let encoded = base64::encode(format!("{}:{}", username, password));
                request.headers_mut().insert(
                    "Authorization",
                    format!("Basic {}", encoded).parse().unwrap(),
                );
            }
        }
        Ok(())
    }
}
```

## Command Pattern with Undo

```rust
pub trait Command {
    type Error;
    fn execute(&mut self) -> Result<(), Self::Error>;
    fn undo(&mut self) -> Result<(), Self::Error>;
}

pub struct CommandHistory {
    commands: Vec<Box<dyn Command<Error = Box<dyn std::error::Error>>>>,
    position: usize,
}

impl CommandHistory {
    pub fn undo(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        if self.position > 0 {
            self.position -= 1;
            self.commands[self.position].undo()?;
        }
        Ok(())
    }

    pub fn redo(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        if self.position < self.commands.len() {
            self.commands[self.position].execute()?;
            self.position += 1;
        }
        Ok(())
    }
}
```

## Observer Pattern with Async

```rust
use tokio::sync::broadcast;

#[derive(Debug, Clone)]
pub enum DomainEvent {
    UserCreated { user_id: UserId, email: String },
    UserUpdated { user_id: UserId, changes: Vec<String> },
    OrderPlaced { order_id: OrderId, user_id: UserId, amount: Decimal },
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

## Adapter Pattern

```rust
#[async_trait::async_trait]
pub trait PaymentProcessor {
    async fn process(&self, payment: &Payment) -> Result<PaymentResult, PaymentError>;
}

pub struct StripeAdapter {
    client: StripeClient,
}

#[async_trait::async_trait]
impl PaymentProcessor for StripeAdapter {
    async fn process(&self, payment: &Payment) -> Result<PaymentResult, PaymentError> {
        let charge = self.client.charge(payment.amount_cents(), &payment.token).await?;
        Ok(PaymentResult { id: charge.id, status: charge.status.into() })
    }
}
```

## Decorator Pattern

```rust
#[async_trait::async_trait]
pub trait HttpHandler {
    async fn handle(&self, request: Request) -> Result<Response, HttpError>;
}

pub struct LoggingDecorator<H: HttpHandler> {
    inner: H,
}

#[async_trait::async_trait]
impl<H: HttpHandler + Send + Sync> HttpHandler for LoggingDecorator<H> {
    async fn handle(&self, request: Request) -> Result<Response, HttpError> {
        let start = std::time::Instant::now();
        tracing::info!("Request: {} {}", request.method(), request.uri());
        let result = self.inner.handle(request).await;
        tracing::info!("Response in {:?}", start.elapsed());
        result
    }
}
```

## Actor Pattern

```rust
use tokio::sync::{mpsc, oneshot};

pub enum UserActorMessage {
    GetUser {
        user_id: UserId,
        respond_to: oneshot::Sender<Result<User, UserError>>,
    },
    UpdateUser {
        user_id: UserId,
        updates: UserUpdates,
        respond_to: oneshot::Sender<Result<User, UserError>>,
    },
}

pub struct UserActor {
    receiver: mpsc::Receiver<UserActorMessage>,
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
    sender: mpsc::Sender<UserActorMessage>,
}

impl UserActorHandle {
    pub async fn get_user(&self, user_id: UserId) -> Result<User, UserError> {
        let (respond_to, response) = oneshot::channel();
        self.sender.send(UserActorMessage::GetUser { user_id, respond_to }).await
            .map_err(|_| UserError::ActorUnavailable)?;
        response.await.map_err(|_| UserError::ActorUnavailable)?
    }
}
```

## Design Patterns Checklist

```markdown
### Design Patterns Verification
- [ ] Builder pattern used for complex configuration
- [ ] Factory pattern for creating related object families
- [ ] Strategy pattern for runtime algorithm selection
- [ ] Command pattern for undo/redo operations
- [ ] Observer pattern for event-driven architecture
- [ ] Adapter pattern for external API integration
- [ ] Decorator pattern for cross-cutting concerns
- [ ] Actor pattern for concurrent state management
- [ ] Type-state pattern for compile-time validation
```

---
*Source: .cursor/rules/rust/core/design-patterns.mdc*
