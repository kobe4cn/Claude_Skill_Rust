---
description: >-
  Use when implementing concurrent or async code in Rust. Provides Tokio patterns
  including sync primitives, DashMap, channels, and graceful shutdown.
  Triggers: async, tokio, concurrency, DashMap, channels, parallel, spawn.
---

# Rust Concurrency Patterns

## Quick Reference

This skill provides guidance for concurrent Rust applications:
- Tokio async runtime and sync primitives
- DashMap for concurrent collections
- Channel patterns (mpsc, broadcast, oneshot)
- Task management and graceful shutdown
- Testing concurrent code

## Key Requirements

### Tokio as Standard
- **Always use Tokio** for async runtime
- Use `tokio::sync` primitives (not std::sync)
- **Never use Mutex<HashMap>** - use DashMap instead
- Avoid blocking operations in async contexts

## Technology Stack

```toml
[dependencies]
tokio = { version = "1.45", features = [
    "macros", "rt-multi-thread", "signal", "sync"
] }
dashmap = { version = "6", features = ["serde"] }
async-trait = "0.1"
futures = "0.3"
tokio-util = "0.7"
```

## Sync Primitives

```rust
// Use tokio::sync, NOT std::sync
use tokio::sync::{RwLock, Mutex, broadcast, mpsc, oneshot};

pub struct UserCache {
    data: Arc<RwLock<HashMap<String, User>>>,
}

impl UserCache {
    pub async fn get(&self, id: &str) -> Option<User> {
        let data = self.data.read().await;
        data.get(id).cloned()
    }
}
```

## DashMap for Concurrent Collections

```rust
use dashmap::DashMap;

// Preferred: DashMap for concurrent access
pub struct ServiceRegistry {
    services: Arc<DashMap<String, Box<dyn Service>>>,
}

impl ServiceRegistry {
    pub fn register(&self, id: String, service: Box<dyn Service>) {
        self.services.insert(id, service);
    }

    pub fn get(&self, id: &str) -> Option<Ref<String, Box<dyn Service>>> {
        self.services.get(id)
    }
}

// NEVER: Mutex<HashMap>
// struct BadRegistry {
//     services: Arc<Mutex<HashMap<String, Service>>>,
// }
```

## Channel Patterns

### MPSC (Multi-Producer Single-Consumer)
```rust
let (tx, mut rx) = mpsc::unbounded_channel();

tokio::spawn(async move {
    while let Some(event) = rx.recv().await {
        process_event(event).await;
    }
});

tx.send(event)?;
```

### Broadcast (Multiple Subscribers)
```rust
let (tx, _) = broadcast::channel(100);

let mut rx = tx.subscribe();
tokio::spawn(async move {
    while let Ok(event) = rx.recv().await {
        handle_event(event).await;
    }
});

tx.send(event)?;
```

### Oneshot (Single Response)
```rust
let (tx, rx) = oneshot::channel();

tokio::spawn(async move {
    let result = compute_result().await;
    let _ = tx.send(result);
});

let result = rx.await?;
```

## Graceful Shutdown

```rust
use tokio_util::sync::CancellationToken;

pub struct Application {
    shutdown_token: CancellationToken,
}

impl Application {
    pub async fn run(&self) {
        let token = self.shutdown_token.clone();

        tokio::spawn(async move {
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
    }

    pub async fn shutdown(&self) {
        self.shutdown_token.cancel();
    }
}
```

## Task Management with JoinSet

```rust
use tokio::task::JoinSet;

let mut join_set = JoinSet::new();

for item in items {
    join_set.spawn(async move {
        process_item(item).await
    });
}

while let Some(result) = join_set.join_next().await {
    handle_result(result?)?;
}
```

## Anti-Patterns to Avoid

- Using `std::sync` in async contexts (blocks runtime)
- Using `parking_lot` in async code
- Using `Mutex<HashMap>` for concurrent access
- Forgetting cancellation in long-running tasks
- Blocking with `std::thread::sleep` in async

## See Also

- `reference.md` - Complete patterns, testing, and performance tips
