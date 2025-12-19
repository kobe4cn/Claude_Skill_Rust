# Rust 可观测性完整参考

## 完整 Tracing 设置

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};
use opentelemetry::trace::TracerProvider;
use opentelemetry_otlp::WithExportConfig;

pub fn setup_tracing(config: &TracingConfig) -> Result<(), TracingError> {
    // 环境过滤器
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new(&config.level));

    // 控制台输出层
    let fmt_layer = tracing_subscriber::fmt::layer()
        .with_target(true)
        .with_thread_ids(true)
        .with_file(true)
        .with_line_number(true);

    // JSON 格式（生产环境）
    let fmt_layer = if config.json_format {
        fmt_layer.json().flatten_event(true).boxed()
    } else {
        fmt_layer.pretty().boxed()
    };

    // OpenTelemetry 层
    let telemetry_layer = if config.otlp_enabled {
        let tracer = opentelemetry_otlp::new_pipeline()
            .tracing()
            .with_exporter(
                opentelemetry_otlp::new_exporter()
                    .tonic()
                    .with_endpoint(&config.otlp_endpoint),
            )
            .install_batch(opentelemetry_sdk::runtime::Tokio)?;

        Some(tracing_opentelemetry::layer().with_tracer(tracer))
    } else {
        None
    };

    tracing_subscriber::registry()
        .with(env_filter)
        .with(fmt_layer)
        .with(telemetry_layer)
        .init();

    Ok(())
}

pub fn shutdown_tracing() {
    opentelemetry::global::shutdown_tracer_provider();
}
```

## 完整 Prometheus 指标

```rust
use prometheus::{
    Registry, Counter, CounterVec, Histogram, HistogramVec, Gauge, GaugeVec,
    Opts, HistogramOpts, Encoder, TextEncoder,
};
use std::sync::Arc;
use dashmap::DashMap;
use std::sync::atomic::{AtomicU64, Ordering};

pub struct MetricsCollector {
    registry: Registry,
    http_requests_total: CounterVec,
    http_request_duration: HistogramVec,
    http_requests_in_flight: Gauge,
    db_connections_active: Gauge,
    db_query_duration: HistogramVec,
    custom_counters: DashMap<String, Counter>,
    custom_gauges: DashMap<String, Gauge>,
}

impl MetricsCollector {
    pub fn new() -> Result<Self, prometheus::Error> {
        let registry = Registry::new();

        // HTTP 指标
        let http_requests_total = CounterVec::new(
            Opts::new("http_requests_total", "Total HTTP requests"),
            &["method", "path", "status"],
        )?;

        let http_request_duration = HistogramVec::new(
            HistogramOpts::new("http_request_duration_seconds", "HTTP request duration")
                .buckets(vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]),
            &["method", "path"],
        )?;

        let http_requests_in_flight = Gauge::new(
            "http_requests_in_flight",
            "Number of HTTP requests currently being processed",
        )?;

        // 数据库指标
        let db_connections_active = Gauge::new(
            "db_connections_active",
            "Number of active database connections",
        )?;

        let db_query_duration = HistogramVec::new(
            HistogramOpts::new("db_query_duration_seconds", "Database query duration")
                .buckets(vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]),
            &["query_type"],
        )?;

        // 注册所有指标
        registry.register(Box::new(http_requests_total.clone()))?;
        registry.register(Box::new(http_request_duration.clone()))?;
        registry.register(Box::new(http_requests_in_flight.clone()))?;
        registry.register(Box::new(db_connections_active.clone()))?;
        registry.register(Box::new(db_query_duration.clone()))?;

        Ok(Self {
            registry,
            http_requests_total,
            http_request_duration,
            http_requests_in_flight,
            db_connections_active,
            db_query_duration,
            custom_counters: DashMap::new(),
            custom_gauges: DashMap::new(),
        })
    }

    /// 记录 HTTP 请求
    pub fn record_request(&self, method: &str, path: &str, status: u16, duration_secs: f64) {
        self.http_requests_total
            .with_label_values(&[method, path, &status.to_string()])
            .inc();
        self.http_request_duration
            .with_label_values(&[method, path])
            .observe(duration_secs);
    }

    /// 增加进行中的请求计数
    pub fn inc_in_flight(&self) {
        self.http_requests_in_flight.inc();
    }

    /// 减少进行中的请求计数
    pub fn dec_in_flight(&self) {
        self.http_requests_in_flight.dec();
    }

    /// 记录数据库查询
    pub fn record_db_query(&self, query_type: &str, duration_secs: f64) {
        self.db_query_duration
            .with_label_values(&[query_type])
            .observe(duration_secs);
    }

    /// 设置活跃连接数
    pub fn set_db_connections(&self, count: i64) {
        self.db_connections_active.set(count as f64);
    }

    /// 获取或创建自定义计数器
    pub fn counter(&self, name: &str) -> Counter {
        self.custom_counters
            .entry(name.to_string())
            .or_insert_with(|| {
                let counter = Counter::new(name, name).unwrap();
                self.registry.register(Box::new(counter.clone())).ok();
                counter
            })
            .clone()
    }

    /// 导出指标
    pub fn export(&self) -> Result<String, prometheus::Error> {
        let encoder = TextEncoder::new();
        let metric_families = self.registry.gather();
        let mut buffer = Vec::new();
        encoder.encode(&metric_families, &mut buffer)?;
        Ok(String::from_utf8(buffer).unwrap())
    }
}
```

## Lock-Free 计数器

```rust
use std::sync::atomic::{AtomicU64, AtomicI64, Ordering};

/// 高性能无锁计数器
#[derive(Debug)]
pub struct AtomicCounter {
    value: AtomicU64,
}

impl AtomicCounter {
    pub fn new() -> Self {
        Self { value: AtomicU64::new(0) }
    }

    pub fn increment(&self) -> u64 {
        self.value.fetch_add(1, Ordering::Relaxed)
    }

    pub fn add(&self, n: u64) -> u64 {
        self.value.fetch_add(n, Ordering::Relaxed)
    }

    pub fn get(&self) -> u64 {
        self.value.load(Ordering::Relaxed)
    }

    pub fn reset(&self) -> u64 {
        self.value.swap(0, Ordering::Relaxed)
    }
}

/// 高性能无锁计量器
#[derive(Debug)]
pub struct AtomicGauge {
    value: AtomicI64,
}

impl AtomicGauge {
    pub fn new() -> Self {
        Self { value: AtomicI64::new(0) }
    }

    pub fn set(&self, value: i64) {
        self.value.store(value, Ordering::Relaxed);
    }

    pub fn inc(&self) {
        self.value.fetch_add(1, Ordering::Relaxed);
    }

    pub fn dec(&self) {
        self.value.fetch_sub(1, Ordering::Relaxed);
    }

    pub fn get(&self) -> i64 {
        self.value.load(Ordering::Relaxed)
    }
}
```

## 请求计时中间件

```rust
use axum::{
    extract::State,
    http::{Request, StatusCode},
    middleware::Next,
    response::Response,
};
use std::time::Instant;

pub async fn metrics_middleware<B>(
    State(metrics): State<Arc<MetricsCollector>>,
    request: Request<B>,
    next: Next<B>,
) -> Response {
    let method = request.method().to_string();
    let path = request.uri().path().to_string();
    let start = Instant::now();

    metrics.inc_in_flight();

    let response = next.run(request).await;

    metrics.dec_in_flight();

    let duration = start.elapsed().as_secs_f64();
    let status = response.status().as_u16();

    metrics.record_request(&method, &path, status, duration);

    // 添加 Server-Timing 头
    let mut response = response;
    response.headers_mut().insert(
        "Server-Timing",
        format!("total;dur={:.3}", duration * 1000.0).parse().unwrap(),
    );

    response
}
```

## 健康检查框架

```rust
use async_trait::async_trait;
use serde::Serialize;
use std::time::Duration;

#[derive(Debug, Clone, Serialize)]
#[serde(rename_all = "lowercase")]
pub enum HealthStatus {
    Healthy,
    Degraded,
    Unhealthy,
}

#[derive(Debug, Clone, Serialize)]
pub struct HealthCheckResult {
    pub name: String,
    pub status: HealthStatus,
    pub message: Option<String>,
    pub duration_ms: u64,
}

#[async_trait]
pub trait HealthCheck: Send + Sync {
    fn name(&self) -> &str;
    async fn check(&self) -> HealthCheckResult;
}

pub struct HealthChecker {
    checks: Vec<Box<dyn HealthCheck>>,
    timeout: Duration,
}

impl HealthChecker {
    pub fn new() -> Self {
        Self {
            checks: Vec::new(),
            timeout: Duration::from_secs(5),
        }
    }

    pub fn add_check<C: HealthCheck + 'static>(mut self, check: C) -> Self {
        self.checks.push(Box::new(check));
        self
    }

    pub fn with_timeout(mut self, timeout: Duration) -> Self {
        self.timeout = timeout;
        self
    }

    pub async fn check_all(&self) -> HealthReport {
        let mut results = Vec::new();
        let mut overall_status = HealthStatus::Healthy;

        for check in &self.checks {
            let result = tokio::time::timeout(self.timeout, check.check())
                .await
                .unwrap_or_else(|_| HealthCheckResult {
                    name: check.name().to_string(),
                    status: HealthStatus::Unhealthy,
                    message: Some("Health check timed out".to_string()),
                    duration_ms: self.timeout.as_millis() as u64,
                });

            match result.status {
                HealthStatus::Unhealthy => overall_status = HealthStatus::Unhealthy,
                HealthStatus::Degraded if !matches!(overall_status, HealthStatus::Unhealthy) => {
                    overall_status = HealthStatus::Degraded;
                }
                _ => {}
            }

            results.push(result);
        }

        HealthReport {
            status: overall_status,
            checks: results,
        }
    }
}

#[derive(Debug, Serialize)]
pub struct HealthReport {
    pub status: HealthStatus,
    pub checks: Vec<HealthCheckResult>,
}

// 数据库健康检查实现
pub struct DatabaseHealthCheck {
    pool: sqlx::PgPool,
}

#[async_trait]
impl HealthCheck for DatabaseHealthCheck {
    fn name(&self) -> &str {
        "database"
    }

    async fn check(&self) -> HealthCheckResult {
        let start = Instant::now();

        let result = sqlx::query("SELECT 1")
            .fetch_one(&self.pool)
            .await;

        let duration_ms = start.elapsed().as_millis() as u64;

        match result {
            Ok(_) => HealthCheckResult {
                name: self.name().to_string(),
                status: if duration_ms > 100 {
                    HealthStatus::Degraded
                } else {
                    HealthStatus::Healthy
                },
                message: None,
                duration_ms,
            },
            Err(e) => HealthCheckResult {
                name: self.name().to_string(),
                status: HealthStatus::Unhealthy,
                message: Some(e.to_string()),
                duration_ms,
            },
        }
    }
}

// Redis 健康检查实现
pub struct RedisHealthCheck {
    client: redis::Client,
}

#[async_trait]
impl HealthCheck for RedisHealthCheck {
    fn name(&self) -> &str {
        "redis"
    }

    async fn check(&self) -> HealthCheckResult {
        let start = Instant::now();

        let result: Result<String, _> = redis::cmd("PING")
            .query_async(&mut self.client.get_multiplexed_async_connection().await.unwrap())
            .await;

        let duration_ms = start.elapsed().as_millis() as u64;

        match result {
            Ok(_) => HealthCheckResult {
                name: self.name().to_string(),
                status: HealthStatus::Healthy,
                message: None,
                duration_ms,
            },
            Err(e) => HealthCheckResult {
                name: self.name().to_string(),
                status: HealthStatus::Unhealthy,
                message: Some(e.to_string()),
                duration_ms,
            },
        }
    }
}
```

## 健康检查端点

```rust
use axum::{Json, extract::State, http::StatusCode};

pub async fn health_handler(
    State(checker): State<Arc<HealthChecker>>,
) -> (StatusCode, Json<HealthReport>) {
    let report = checker.check_all().await;

    let status_code = match report.status {
        HealthStatus::Healthy => StatusCode::OK,
        HealthStatus::Degraded => StatusCode::OK,
        HealthStatus::Unhealthy => StatusCode::SERVICE_UNAVAILABLE,
    };

    (status_code, Json(report))
}

pub async fn liveness_handler() -> StatusCode {
    StatusCode::OK
}

pub async fn readiness_handler(
    State(checker): State<Arc<HealthChecker>>,
) -> StatusCode {
    let report = checker.check_all().await;

    match report.status {
        HealthStatus::Healthy | HealthStatus::Degraded => StatusCode::OK,
        HealthStatus::Unhealthy => StatusCode::SERVICE_UNAVAILABLE,
    }
}

// 指标端点
pub async fn metrics_handler(
    State(metrics): State<Arc<MetricsCollector>>,
) -> Result<String, StatusCode> {
    metrics.export().map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)
}
```

## 分布式追踪上下文传播

```rust
use opentelemetry::propagation::{TextMapPropagator, Extractor, Injector};
use opentelemetry_sdk::propagation::TraceContextPropagator;
use tracing_opentelemetry::OpenTelemetrySpanExt;

struct HeaderExtractor<'a>(&'a http::HeaderMap);

impl<'a> Extractor for HeaderExtractor<'a> {
    fn get(&self, key: &str) -> Option<&str> {
        self.0.get(key).and_then(|v| v.to_str().ok())
    }

    fn keys(&self) -> Vec<&str> {
        self.0.keys().map(|k| k.as_str()).collect()
    }
}

struct HeaderInjector<'a>(&'a mut http::HeaderMap);

impl<'a> Injector for HeaderInjector<'a> {
    fn set(&mut self, key: &str, value: String) {
        if let Ok(name) = http::header::HeaderName::from_bytes(key.as_bytes()) {
            if let Ok(val) = http::header::HeaderValue::from_str(&value) {
                self.0.insert(name, val);
            }
        }
    }
}

// 从请求提取追踪上下文
pub fn extract_trace_context(headers: &http::HeaderMap) -> opentelemetry::Context {
    let propagator = TraceContextPropagator::new();
    propagator.extract(&HeaderExtractor(headers))
}

// 注入追踪上下文到请求
pub fn inject_trace_context(headers: &mut http::HeaderMap) {
    let propagator = TraceContextPropagator::new();
    let context = tracing::Span::current().context();
    propagator.inject_context(&context, &mut HeaderInjector(headers));
}
```

## 测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_atomic_counter() {
        let counter = AtomicCounter::new();
        assert_eq!(counter.get(), 0);

        counter.increment();
        assert_eq!(counter.get(), 1);

        counter.add(10);
        assert_eq!(counter.get(), 11);

        let old = counter.reset();
        assert_eq!(old, 11);
        assert_eq!(counter.get(), 0);
    }

    #[tokio::test]
    async fn test_health_checker() {
        struct AlwaysHealthy;

        #[async_trait]
        impl HealthCheck for AlwaysHealthy {
            fn name(&self) -> &str { "always_healthy" }
            async fn check(&self) -> HealthCheckResult {
                HealthCheckResult {
                    name: self.name().to_string(),
                    status: HealthStatus::Healthy,
                    message: None,
                    duration_ms: 1,
                }
            }
        }

        let checker = HealthChecker::new().add_check(AlwaysHealthy);
        let report = checker.check_all().await;

        assert!(matches!(report.status, HealthStatus::Healthy));
        assert_eq!(report.checks.len(), 1);
    }

    #[test]
    fn test_metrics_export() {
        let metrics = MetricsCollector::new().unwrap();
        metrics.record_request("GET", "/api/users", 200, 0.05);

        let output = metrics.export().unwrap();
        assert!(output.contains("http_requests_total"));
        assert!(output.contains("http_request_duration_seconds"));
    }
}
```

## 检查清单

```markdown
### 可观测性检查清单
- [ ] 结构化日志（JSON 格式生产环境）
- [ ] 请求追踪 ID 贯穿整个请求
- [ ] Prometheus 指标端点 /metrics
- [ ] 健康检查端点 /health, /ready, /live
- [ ] 所有外部调用都有超时和重试指标
- [ ] 数据库查询时间指标
- [ ] 使用 lock-free 计数器
- [ ] 分布式追踪上下文传播
- [ ] 错误率指标
- [ ] 资源使用指标（连接池、内存）
```
