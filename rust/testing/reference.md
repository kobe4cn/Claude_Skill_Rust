# Rust Testing Reference

## Project Structure

```
my_project/
├── Cargo.toml
├── src/
│   ├── lib.rs           # Library with unit tests
│   ├── main.rs          # Binary entry point
│   └── user/
│       ├── mod.rs       # Module with inline tests
│       ├── service.rs   # Service with unit tests
│       └── repository.rs
├── tests/               # Integration tests
│   ├── integration_test.rs
│   ├── api_test.rs
│   └── common/
│       └── mod.rs
└── benches/             # Benchmarks
    └── user_benchmarks.rs
```

## Unit Testing with Mocking

```rust
use mockall::{mock, predicate::*};

// Define trait for mocking
#[async_trait::async_trait]
pub trait UserRepository {
    async fn save(&self, user: User) -> Result<User, DbError>;
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, DbError>;
}

// Generate mock
mock! {
    pub TestUserRepository {}

    #[async_trait::async_trait]
    impl UserRepository for TestUserRepository {
        async fn save(&self, user: User) -> Result<User, DbError>;
        async fn find_by_id(&self, id: &str) -> Result<Option<User>, DbError>;
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_create_user_success() {
        let mut mock_repo = MockTestUserRepository::new();
        let user = User {
            id: "123".to_string(),
            email: "test@example.com".to_string(),
            name: "Test User".to_string(),
        };

        mock_repo
            .expect_save()
            .with(eq(user.clone()))
            .times(1)
            .returning(|user| Ok(user));

        let service = UserService::new(mock_repo);
        let result = service.create_user(user.clone()).await;

        assert!(result.is_ok());
        assert_eq!(result.unwrap(), user);
    }

    #[tokio::test]
    async fn test_create_user_empty_email_fails() {
        let mock_repo = MockTestUserRepository::new();
        let user = User {
            id: "123".to_string(),
            email: "".to_string(),
            name: "Test User".to_string(),
        };

        let service = UserService::new(mock_repo);
        let result = service.create_user(user).await;

        assert!(result.is_err());
        assert!(matches!(result.unwrap_err(), UserError::InvalidInput(_)));
    }
}
```

## Integration Testing with Testcontainers

```rust
// tests/integration_test.rs
use testcontainers::{clients::Cli, images::postgres::Postgres, Container};
use sqlx::PgPool;

struct TestContext {
    pool: PgPool,
    _container: Container<'static, Postgres>,
}

impl TestContext {
    async fn new() -> Self {
        let docker = Cli::default();
        let container = docker.run(Postgres::default());
        let host_port = container.get_host_port_ipv4(5432);

        let database_url = format!(
            "postgres://postgres:password@localhost:{}/test",
            host_port
        );
        let pool = PgPool::connect(&database_url).await.unwrap();

        // Run migrations
        sqlx::migrate!("./migrations").run(&pool).await.unwrap();

        Self { pool, _container: container }
    }
}

#[tokio::test]
async fn test_user_crud_operations() {
    let ctx = TestContext::new().await;
    let repository = PostgresUserRepository::new(ctx.pool.clone());
    let service = UserService::new(repository);

    // Create
    let user = User::new("test@example.com", "Test User");
    let created = service.create_user(user.clone()).await.unwrap();
    assert_eq!(created.email, user.email);

    // Read
    let found = service.find_user(&created.id).await.unwrap();
    assert!(found.is_some());

    // Update
    let updated = service.update_user(&created.id, "New Name").await.unwrap();
    assert_eq!(updated.name, "New Name");

    // Delete
    service.delete_user(&created.id).await.unwrap();
    let deleted = service.find_user(&created.id).await.unwrap();
    assert!(deleted.is_none());
}
```

## Property Testing

```rust
use proptest::prelude::*;

pub fn validate_email(email: &str) -> bool {
    email.contains('@') && email.len() > 3 && email.len() < 255
}

pub fn validate_username(username: &str) -> bool {
    username.len() >= 3 && username.len() <= 50
        && username.chars().all(|c| c.is_alphanumeric() || c == '_')
}

#[cfg(test)]
mod property_tests {
    use super::*;

    proptest! {
        #[test]
        fn test_valid_email(email in r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}") {
            prop_assert!(validate_email(&email));
        }

        #[test]
        fn test_invalid_email_without_at(email in r"[a-zA-Z0-9._%+-]+[a-zA-Z0-9.-]+") {
            prop_assume!(!email.contains('@'));
            prop_assert!(!validate_email(&email));
        }

        #[test]
        fn test_username_length_bounds(username in r"[a-zA-Z0-9_]{3,50}") {
            prop_assert!(validate_username(&username));
        }

        #[test]
        fn test_user_roundtrip(user in valid_user_strategy()) {
            let json = serde_json::to_string(&user).unwrap();
            let deserialized: User = serde_json::from_str(&json).unwrap();
            prop_assert_eq!(user, deserialized);
        }
    }

    fn valid_user_strategy() -> impl Strategy<Value = User> {
        (
            r"user[0-9]{1,6}",
            r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",
            r"[A-Z][a-z]{2,20} [A-Z][a-z]{2,20}",
        ).prop_map(|(id, email, name)| User { id, email, name })
    }
}
```

## Web API Testing

```rust
// tests/api_test.rs
use axum_test::TestServer;
use wiremock::{MockServer, Mock, ResponseTemplate};
use wiremock::matchers::{method, path};

#[tokio::test]
async fn test_create_user_api() {
    let app_state = AppState::new_test();
    let app = create_app(app_state);
    let server = TestServer::new(app).unwrap();

    let user_data = serde_json::json!({
        "email": "api@example.com",
        "name": "API User"
    });

    let response = server
        .post("/users")
        .json(&user_data)
        .await;

    response.assert_status_created();
    let body: serde_json::Value = response.json();
    assert_eq!(body["email"], "api@example.com");
}

#[tokio::test]
async fn test_external_service_mock() {
    let mock_server = MockServer::start().await;

    Mock::given(method("GET"))
        .and(path("/external/users/123"))
        .respond_with(ResponseTemplate::new(200)
            .set_body_json(serde_json::json!({
                "id": "123",
                "verified": true
            })))
        .mount(&mock_server)
        .await;

    let app_state = AppState::new_test_with_url(&mock_server.uri());
    let app = create_app(app_state);
    let server = TestServer::new(app).unwrap();

    let response = server.get("/users/123/verification").await;
    response.assert_status_ok();
}
```

## Benchmarking

```rust
// benches/user_benchmarks.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};

fn create_user_benchmark(c: &mut Criterion) {
    let repository = InMemoryUserRepository::new();
    let service = UserService::new(repository);

    c.bench_function("create_user", |b| {
        b.iter(|| {
            let user = User {
                id: format!("bench-{}", fastrand::u64(..)),
                email: "bench@example.com".to_string(),
                name: "Benchmark User".to_string(),
            };
            black_box(service.create_user(user))
        })
    });
}

fn find_user_benchmark(c: &mut Criterion) {
    let mut group = c.benchmark_group("find_user");

    for size in [100, 1000, 10000] {
        let repository = InMemoryUserRepository::new();
        let service = UserService::new(repository);

        // Pre-populate
        for i in 0..size {
            let user = User {
                id: format!("user-{}", i),
                email: format!("user{}@example.com", i),
                name: format!("User {}", i),
            };
            service.create_user(user).unwrap();
        }

        group.bench_with_input(BenchmarkId::new("size", size), &size, |b, &size| {
            b.iter(|| {
                let id = format!("user-{}", fastrand::usize(0..size));
                black_box(service.find_user(&id))
            })
        });
    }
    group.finish();
}

criterion_group!(benches, create_user_benchmark, find_user_benchmark);
criterion_main!(benches);
```

## GitHub Actions CI

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
          POSTGRES_DB: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
    - uses: actions/checkout@v4

    - name: Install Rust
      uses: dtolnay/rust-toolchain@stable
      with:
        components: rustfmt, clippy

    - name: Cache
      uses: actions/cache@v4
      with:
        path: |
          ~/.cargo/registry/
          ~/.cargo/git/
          target/
        key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

    - name: Check formatting
      run: cargo fmt --all -- --check

    - name: Clippy
      run: cargo clippy --all-targets -- -D warnings

    - name: Unit Tests
      run: cargo test --lib

    - name: Integration Tests
      run: cargo test --test '*'
      env:
        DATABASE_URL: postgres://postgres:password@localhost:5432/test

    - name: Benchmarks (check only)
      run: cargo bench --no-run
```

## Testing Checklist

```markdown
### Testing Verification
- [ ] Unit tests for all public functions
- [ ] Unit tests use mocking for dependencies
- [ ] Integration tests with testcontainers
- [ ] Property tests validate invariants
- [ ] Benchmarks track performance
- [ ] Test coverage > 80% for core modules
- [ ] Tests run in CI/CD pipeline
- [ ] Async tests use `#[tokio::test]`
- [ ] Web API tests cover error cases
- [ ] External services are mocked
```

---
*Source: .cursor/rules/rust/quality/testing.mdc*
