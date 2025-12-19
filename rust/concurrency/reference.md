# Rust Concurrency Reference

## Sync Primitives Comparison

| Use Case | Recommended | Avoid |
|----------|-------------|-------|
| Async RwLock | `tokio::sync::RwLock` | `std::sync::RwLock` |
| Async Mutex | `tokio::sync::Mutex` | `std::sync::Mutex` |
| Concurrent HashMap | `DashMap` | `Mutex<HashMap>` |
| One-shot response | `tokio::sync::oneshot` | Manual channel |
| Multiple subscribers | `broadcast` | Multiple mpsc |

## Complete DashMap Example

```rust
use dashmap::DashMap;
use std::sync::Arc;

pub struct ServiceRegistry {
    services: Arc<DashMap<String, Box<dyn Service + Send + Sync>>>,
    categories: Arc<DashMap<String, Vec<String>>>,
}

impl ServiceRegistry {
    pub fn new() -> Self {
        Self {
            services: Arc::new(DashMap::new()),
            categories: Arc::new(DashMap::new()),
        }
    }

    pub fn register(&self, id: String, service: Box<dyn Service + Send + Sync>) {
        let category = service.category().to_string();
        self.services.insert(id.clone(), service);
        self.categories
            .entry(category)
            .or_insert_with(Vec::new)
            .push(id);
    }

    pub fn get(&self, id: &str) -> Option<dashmap::mapref::one::Ref<String, Box<dyn Service + Send + Sync>>> {
        self.services.get(id)
    }

    pub fn list_all(&self) -> Vec<String> {
        self.services.iter().map(|e| e.key().clone()).collect()
    }
}
```

## Channel Patterns

### MPSC Event Processor

```rust
use tokio::sync::mpsc;

pub struct EventProcessor {
    sender: mpsc::UnboundedSender<SystemEvent>,
}

pub struct EventProcessorHandle {
    receiver: mpsc::UnboundedReceiver<SystemEvent>,
}

impl EventProcessor {
    pub fn new() -> (Self, EventProcessorHandle) {
        let (tx, rx) = mpsc::unbounded_channel();
        (Self { sender: tx }, EventProcessorHandle { receiver: rx })
    }

    pub fn send(&self, event: SystemEvent) -> Result<(), mpsc::error::SendError<SystemEvent>> {
        self.sender.send(event)
    }
}

impl EventProcessorHandle {
    pub async fn run(mut self) {
        while let Some(event) = self.receiver.recv().await {
            if let Err(e) = self.process(event).await {
                tracing::error!("Failed: {}", e);
            }
        }
    }

    async fn process(&self, event: SystemEvent) -> Result<(), ProcessingError> {
        match event {
            SystemEvent::UserRegistered { user_id, .. } => {
                tracing::info!("User {} registered", user_id);
            }
            // Handle other events...
        }
        Ok(())
    }
}
```

### Broadcast Event Bus

```rust
use tokio::sync::broadcast;

pub struct EventBus {
    sender: broadcast::Sender<SystemEvent>,
}

impl EventBus {
    pub fn new(capacity: usize) -> Self {
        let (sender, _) = broadcast::channel(capacity);
        Self { sender }
    }

    pub fn publish(&self, event: SystemEvent) -> Result<usize, broadcast::error::SendError<SystemEvent>> {
        self.sender.send(event)
    }

    pub fn subscribe(&self) -> broadcast::Receiver<SystemEvent> {
        self.sender.subscribe()
    }
}

// Usage
pub async fn start_monitoring(bus: Arc<EventBus>) {
    let mut rx = bus.subscribe();

    tokio::spawn(async move {
        while let Ok(event) = rx.recv().await {
            handle_event(event).await;
        }
    });
}
```

### Oneshot Request-Response

```rust
use tokio::sync::oneshot;

pub struct AsyncValidator;

impl AsyncValidator {
    pub async fn validate(&self, user: User) -> Result<ValidationResult, ValidationError> {
        let (tx, rx) = oneshot::channel();

        tokio::spawn(async move {
            let result = perform_validation(user).await;
            let _ = tx.send(result);
        });

        rx.await
            .map_err(|_| ValidationError::Cancelled)?
    }
}
```

## Task Management

### JoinSet for Parallel Tasks

```rust
use tokio::task::JoinSet;

pub async fn process_batch(items: &[Item]) -> Result<Vec<ItemResult>, ProcessingError> {
    let mut join_set = JoinSet::new();
    let mut results = Vec::new();

    for item in items {
        let item = item.clone();
        join_set.spawn(async move {
            process_item(item).await
        });
    }

    while let Some(result) = join_set.join_next().await {
        match result {
            Ok(item_result) => results.push(item_result?),
            Err(e) => return Err(ProcessingError::TaskFailed(e.to_string())),
        }
    }

    Ok(results)
}
```

### Graceful Shutdown

```rust
use tokio_util::sync::CancellationToken;

pub struct Application {
    shutdown_token: CancellationToken,
    tasks: Vec<tokio::task::JoinHandle<()>>,
}

impl Application {
    pub fn new() -> Self {
        Self {
            shutdown_token: CancellationToken::new(),
            tasks: Vec::new(),
        }
    }

    pub async fn start(&mut self) -> Result<(), ApplicationError> {
        self.start_background_service().await?;
        self.wait_for_shutdown().await;
        self.shutdown_gracefully().await
    }

    async fn start_background_service(&mut self) -> Result<(), ApplicationError> {
        let token = self.shutdown_token.clone();

        let handle = tokio::spawn(async move {
            loop {
                tokio::select! {
                    _ = token.cancelled() => {
                        tracing::info!("Shutdown requested");
                        break;
                    }
                    _ = do_work() => {}
                }
            }
        });

        self.tasks.push(handle);
        Ok(())
    }

    async fn wait_for_shutdown(&self) {
        let mut sigterm = tokio::signal::unix::signal(
            tokio::signal::unix::SignalKind::terminate()
        ).unwrap();

        tokio::select! {
            _ = sigterm.recv() => tracing::info!("SIGTERM received"),
            _ = tokio::signal::ctrl_c() => tracing::info!("SIGINT received"),
        }

        self.shutdown_token.cancel();
    }

    async fn shutdown_gracefully(&mut self) -> Result<(), ApplicationError> {
        let timeout = tokio::time::Duration::from_secs(30);

        tokio::time::timeout(timeout, async {
            for handle in self.tasks.drain(..) {
                let _ = handle.await;
            }
        }).await.map_err(|_| ApplicationError::ShutdownTimeout)?;

        Ok(())
    }
}
```

## Testing Concurrent Code

```rust
#[cfg(test)]
mod tests {
    use tokio::time::{timeout, Duration};

    #[tokio::test]
    async fn test_concurrent_access() {
        let cache = UserCache::new();
        let mut handles = Vec::new();

        for i in 0..10 {
            let cache = cache.clone();
            handles.push(tokio::spawn(async move {
                cache.insert(format!("user_{}", i), User::default()).await;
            }));
        }

        for handle in handles {
            handle.await.unwrap();
        }

        for i in 0..10 {
            assert!(cache.get(&format!("user_{}", i)).await.is_some());
        }
    }

    #[tokio::test]
    async fn test_with_timeout() {
        let result = timeout(Duration::from_secs(5), async {
            // Long-running operation
            tokio::time::sleep(Duration::from_millis(100)).await;
            "done"
        }).await;

        assert!(result.is_ok());
    }
}
```

## Concurrency Checklist

```markdown
### Sync Primitives
- [ ] Uses tokio::sync (not std::sync)
- [ ] DashMap for concurrent collections
- [ ] No Mutex<HashMap> anywhere

### Channel Usage
- [ ] Appropriate channel type selected
- [ ] Bounded channels where needed
- [ ] Proper error handling on send/recv

### Task Management
- [ ] Long-running tasks support cancellation
- [ ] JoinSet for parallel task groups
- [ ] Graceful shutdown implemented

### Testing
- [ ] Concurrent access tests
- [ ] Timeouts on potentially hanging operations
- [ ] No unwrap/expect in concurrent code
```

---
*Source: cursor-rust-rules features/concurrency.mdc*
