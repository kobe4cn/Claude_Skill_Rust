---
description: >-
  Use when building web APIs with Axum framework. Provides Axum 0.8+ patterns
  including AppState, OpenAPI with utoipa, middleware, and router organization.
  Triggers: Axum, web server, HTTP API, REST API, OpenAPI, Swagger.
---

# Rust Axum Web Framework

## Quick Reference

This skill provides guidance for building production-ready web APIs with Axum 0.8+:
- AppState and AppConfig patterns with arc-swap
- OpenAPI integration with utoipa
- Router organization and middleware
- Request/response handling
- Error handling patterns

## Key Requirements

### Axum 0.8+ Standards
- Use `{param}` path syntax (not `:param`)
- Structured router organization
- OpenAPI integration with utoipa
- AppState pattern with arc-swap for hot-reloadable config

## Technology Stack

```toml
[dependencies]
axum = { version = "0.8", features = ["macros", "multipart"] }
tokio = { version = "1.45", features = ["macros", "rt-multi-thread", "signal"] }
tower-http = { version = "0.6", features = ["cors", "trace", "compression"] }
arc-swap = "1.0"
utoipa = { version = "5.0", features = ["axum_extras", "chrono", "uuid"] }
utoipa-swagger-ui = { version = "9.0", features = ["axum"] }
validator = { version = "0.18", features = ["derive"] }
```

## AppState Pattern

```rust
use arc_swap::ArcSwap;
use std::sync::Arc;

#[derive(Clone)]
pub struct AppState {
    pub config: Arc<ArcSwap<AppConfig>>,
    pub db: PgPool,
    pub http_client: reqwest::Client,
}

impl AppState {
    pub fn config(&self) -> Arc<AppConfig> {
        self.config.load_full()
    }

    pub fn update_config(&self, new_config: AppConfig) {
        self.config.store(Arc::new(new_config));
    }
}
```

## Router Organization

```rust
pub fn create_app(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health_check))
        .nest("/api/v1", api_routes())
        .layer(TraceLayer::new_for_http())
        .layer(CompressionLayer::new())
        .with_state(state)
}

fn api_routes() -> Router<AppState> {
    Router::new()
        .nest("/users", users::routes())
        .nest("/products", products::routes())
}
```

## OpenAPI Integration

```rust
#[utoipa::path(
    get,
    path = "/api/v1/users/{user_id}",
    params(("user_id" = Uuid, Path, description = "User ID")),
    responses(
        (status = 200, description = "User details", body = User),
        (status = 404, description = "User not found", body = ApiError)
    ),
    tag = "users"
)]
pub async fn get_user(
    State(state): State<AppState>,
    Path(user_id): Path<Uuid>,
) -> Result<Json<User>, ApiError> {
    // Implementation
}
```

## Error Handling

```rust
#[derive(Debug, Serialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct ApiError {
    pub code: String,
    pub message: String,
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let status = match self.code.as_str() {
            "NOT_FOUND" => StatusCode::NOT_FOUND,
            "UNAUTHORIZED" => StatusCode::UNAUTHORIZED,
            _ => StatusCode::INTERNAL_SERVER_ERROR,
        };
        (status, Json(self)).into_response()
    }
}
```

## Anti-Patterns to Avoid

- Old path syntax (`:param` instead of `{param}`)
- Missing request validation
- Blocking operations in async handlers
- Missing OpenAPI documentation
- Unstructured error handling

## See Also

- `reference.md` - Complete patterns, middleware, and testing examples
