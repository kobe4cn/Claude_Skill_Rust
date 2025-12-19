---
description: >-
  Use when implementing observability in Rust applications. Provides Prometheus metrics,
  OpenTelemetry tracing, and health check patterns.
  Triggers: metrics, tracing, Prometheus, OpenTelemetry, health checks, monitoring.
---

# Rust Observability Patterns

## Quick Reference

This skill provides guidance for observability:
- Lock-free metrics with Prometheus
- Distributed tracing with OpenTelemetry
- Health check framework
- Performance monitoring patterns

## Key Requirements

### Observability Standards
- Use lock-free atomic counters for high-throughput
- DashMap for concurrent metrics collection
- OpenTelemetry for distributed tracing
- Comprehensive health checks for all dependencies

## Technology Stack

```toml
[dependencies]
prometheus = "0.13"
opentelemetry = "0.22"
opentelemetry-jaeger = "0.22"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
tracing-opentelemetry = "0.23"
dashmap = "6.0"
```

## Lock-Free Metrics

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use dashmap::DashMap;

pub struct AtomicCounter {
    value: AtomicU64,
}

impl AtomicCounter {
    pub fn increment(&self) -> u64 {
        self.value.fetch_add(1, Ordering::Relaxed)
    }

    pub fn get(&self) -> u64 {
        self.value.load(Ordering::Relaxed)
    }
}

pub struct MetricsCollector {
    counters: DashMap<String, Arc<AtomicCounter>>,
    registry: prometheus::Registry,
}
```

## Request Timing

```rust
pub struct RequestTimer {
    start_time: Instant,
    metrics: Arc<MetricsCollector>,
    operation: String,
}

impl RequestTimer {
    pub fn start(metrics: Arc<MetricsCollector>, operation: &str) -> Self {
        metrics.counter(&format!("{}_requests_total", operation)).increment();
        Self { start_time: Instant::now(), metrics, operation: operation.into() }
    }

    pub fn finish(self) -> Duration {
        let duration = self.start_time.elapsed();
        // Record to histogram
        duration
    }
}
```

## Health Checks

```rust
#[derive(Debug, Clone, Serialize)]
pub enum HealthStatus {
    Healthy,
    Degraded,
    Unhealthy,
}

#[async_trait]
pub trait HealthCheck: Send + Sync {
    async fn check(&self) -> HealthCheckResult;
    fn name(&self) -> &str;
}

pub struct DatabaseHealthCheck {
    pool: PgPool,
}

#[async_trait]
impl HealthCheck for DatabaseHealthCheck {
    async fn check(&self) -> HealthCheckResult {
        match sqlx::query("SELECT 1").fetch_one(&self.pool).await {
            Ok(_) => HealthCheckResult::healthy("Database connected"),
            Err(e) => HealthCheckResult::unhealthy(e.to_string()),
        }
    }
}
```

## Distributed Tracing

```rust
use tracing::{info_span, Span};
use tracing_opentelemetry::OpenTelemetrySpanExt;

pub fn setup_tracing(service_name: &str) -> Result<(), TracingError> {
    let tracer = opentelemetry_jaeger::new_agent_pipeline()
        .with_service_name(service_name)
        .install_simple()?;

    let telemetry = tracing_opentelemetry::layer().with_tracer(tracer);

    tracing_subscriber::registry()
        .with(telemetry)
        .with(tracing_subscriber::fmt::layer())
        .init();

    Ok(())
}
```

## Anti-Patterns to Avoid

- Mutex-based metrics (use atomics)
- Blocking health checks without timeout
- Missing trace context propagation
- No error rate metrics
- Health endpoints without dependency checks

## See Also

- `reference.md` - Complete patterns, exporters, and testing
