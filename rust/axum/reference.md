# Rust Axum Reference

## Dependencies Configuration

```toml
[dependencies]
axum = { version = "0.8", features = ["macros", "multipart"] }
tokio = { version = "1.45", features = ["macros", "rt-multi-thread", "net", "fs", "time", "sync", "signal"] }
tower = { version = "0.5", features = ["full"] }
tower-http = { version = "0.6", features = ["cors", "trace", "compression", "auth", "limit"] }
http = "1.0"
arc-swap = "1.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
utoipa = { version = "5.0", features = ["axum_extras", "chrono", "uuid"] }
utoipa-axum = "0.2"
utoipa-swagger-ui = { version = "9.0", features = ["axum"] }
anyhow = "1.0"
thiserror = "2.0"
validator = { version = "0.18", features = ["derive"] }
uuid = { version = "1.17", features = ["v4", "v7", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
tracing = "0.1"
sqlx = { version = "0.8", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }

[dev-dependencies]
axum-test = "15.0"
tower-test = "0.4"
wiremock = "0.6"
```

## Complete AppConfig Example

```rust
use arc_swap::ArcSwap;
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use utoipa::ToSchema;

#[derive(Debug, Clone, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct AppConfig {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub auth: AuthConfig,
    pub features: FeatureFlags,
}

#[derive(Debug, Clone, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
    pub cors_origins: Vec<String>,
    pub request_timeout_seconds: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct FeatureFlags {
    pub enable_registration: bool,
    pub enable_swagger: bool,
    pub enable_metrics: bool,
}
```

## Complete Router Setup

```rust
#[derive(OpenApi)]
#[openapi(
    paths(
        routes::health::health_check,
        routes::auth::login,
        routes::users::get_user,
        routes::users::list_users,
    ),
    components(schemas(User, ApiError, LoginRequest, TokenResponse)),
    tags(
        (name = "health", description = "Health check endpoints"),
        (name = "auth", description = "Authentication endpoints"),
        (name = "users", description = "User management endpoints"),
    ),
    info(title = "My API", version = "1.0.0")
)]
struct ApiDoc;

pub async fn create_app(state: AppState) -> Router {
    let config = state.config();

    let mut router = Router::new()
        .route("/health", get(routes::health::health_check))
        .nest("/api/v1", api_routes())
        .layer(TraceLayer::new_for_http())
        .layer(CompressionLayer::new())
        .layer(cors_layer(&config))
        .with_state(state);

    if config.features.enable_swagger {
        router = router.merge(
            SwaggerUi::new("/swagger-ui")
                .url("/api-docs/openapi.json", ApiDoc::openapi())
        );
    }

    router
}
```

## Middleware Implementation

```rust
use axum::{extract::{Request, State}, http::header::AUTHORIZATION, middleware::Next, response::Response};

pub async fn auth_middleware(
    State(state): State<AppState>,
    mut request: Request,
    next: Next,
) -> Result<Response, ApiError> {
    // Skip auth for public endpoints
    if request.uri().path().starts_with("/health") {
        return Ok(next.run(request).await);
    }

    let auth_header = request
        .headers()
        .get(AUTHORIZATION)
        .and_then(|h| h.to_str().ok())
        .ok_or_else(|| ApiError::unauthorized("Missing authorization"))?;

    let token = auth_header
        .strip_prefix("Bearer ")
        .ok_or_else(|| ApiError::unauthorized("Invalid format"))?;

    let user_id = validate_jwt(token, &state)?;
    request.extensions_mut().insert(user_id);

    Ok(next.run(request).await)
}
```

## Request Validation

```rust
use validator::{Validate, ValidationError};

#[derive(Debug, Deserialize, Validate, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct CreateUserRequest {
    #[validate(email(message = "Invalid email"))]
    pub email: String,

    #[validate(length(min = 8))]
    pub password: String,

    #[validate(length(min = 2, max = 50))]
    pub name: String,
}

pub async fn create_user(
    State(state): State<AppState>,
    Json(request): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<User>), ApiError> {
    request.validate()
        .map_err(|e| ApiError::validation_error(e.to_string()))?;

    // Create user...
    Ok((StatusCode::CREATED, Json(user)))
}
```

## Integration Testing

```rust
#[cfg(test)]
mod tests {
    use axum_test::TestServer;
    use serde_json::json;

    async fn create_test_app() -> TestServer {
        let state = AppState::new(AppConfig::default()).await.unwrap();
        let app = create_app(state).await;
        TestServer::new(app).unwrap()
    }

    #[tokio::test]
    async fn test_create_user() {
        let server = create_test_app().await;

        let response = server
            .post("/api/v1/users")
            .json(&json!({
                "email": "test@example.com",
                "password": "password123",
                "name": "Test User"
            }))
            .await;

        response.assert_status_created();
    }

    #[tokio::test]
    async fn test_validation_error() {
        let server = create_test_app().await;

        let response = server
            .post("/api/v1/users")
            .json(&json!({"email": "invalid", "password": "short"}))
            .await;

        response.assert_status_bad_request();
    }
}
```

## Axum Checklist

```markdown
### Configuration Management
- [ ] AppConfig struct with proper serialization
- [ ] arc-swap for hot-reloadable config
- [ ] Environment variable loading
- [ ] Default configuration provided

### Application State
- [ ] AppState contains all shared resources
- [ ] Database pool properly initialized
- [ ] HTTP client configured with timeouts

### OpenAPI Integration
- [ ] utoipa derives on all types
- [ ] API paths documented with #[utoipa::path]
- [ ] Swagger UI conditionally enabled

### Route Organization
- [ ] Routes organized by feature modules
- [ ] Uses Axum 0.8+ with {param} syntax
- [ ] Request/response types properly defined

### Middleware
- [ ] Authentication middleware implemented
- [ ] CORS layer configured
- [ ] Tracing layer for logging
- [ ] Compression middleware

### Error Handling
- [ ] Structured error types with HTTP status
- [ ] IntoResponse implemented
- [ ] Consistent error format

### Testing
- [ ] Integration tests with TestServer
- [ ] Request/response validation tests
- [ ] Authentication tests
```

---
*Source: cursor-rust-rules features/axum.mdc*
