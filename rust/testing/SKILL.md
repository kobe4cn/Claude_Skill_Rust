---
description: >-
  Use when writing tests for Rust projects. Covers unit testing, integration testing
  with testcontainers, property testing with proptest, benchmarking with criterion,
  mocking with mockall, and CI/CD integration.
  Triggers: test, unit test, integration test, benchmark, mock, proptest, criterion.
---

# Rust Testing Standards

## Quick Reference

Comprehensive testing guidelines covering all testing strategies for Rust projects.

## Unit Testing

```rust
// Place unit tests in the same file as implementation
#[cfg(test)]
mod tests {
    use super::*;
    use mockall::predicate::*;

    #[test]
    fn test_user_validation_success() {
        let user = User {
            id: "123".to_string(),
            email: "test@example.com".to_string(),
            name: "Test User".to_string(),
        };

        assert!(validate_user(&user).is_ok());
    }

    #[test]
    fn test_empty_email_fails() {
        let user = User {
            id: "123".to_string(),
            email: "".to_string(),
            name: "Test".to_string(),
        };

        assert!(matches!(
            validate_user(&user),
            Err(ValidationError::EmptyEmail)
        ));
    }

    #[tokio::test]
    async fn test_async_user_creation() {
        let mut mock_repo = MockUserRepository::new();
        mock_repo
            .expect_save()
            .times(1)
            .returning(|user| Ok(user));

        let service = UserService::new(mock_repo);
        let result = service.create_user("test@example.com").await;

        assert!(result.is_ok());
    }
}
```

## Property Testing with Proptest

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_email_validation(email in r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}") {
        prop_assert!(validate_email(&email).is_ok());
    }

    #[test]
    fn test_serialization_roundtrip(user in valid_user_strategy()) {
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
```

## Integration Testing

```rust
// tests/integration_test.rs
use testcontainers::{clients::Cli, images::postgres::Postgres};

#[tokio::test]
async fn test_user_service_integration() {
    let docker = Cli::default();
    let container = docker.run(Postgres::default());
    let port = container.get_host_port_ipv4(5432);

    let database_url = format!("postgres://postgres:password@localhost:{}/test", port);
    let pool = sqlx::PgPool::connect(&database_url).await.unwrap();

    sqlx::migrate!("./migrations").run(&pool).await.unwrap();

    let repo = PostgresUserRepository::new(pool);
    let service = UserService::new(repo);

    let user = service.create_user("test@example.com").await.unwrap();
    assert!(service.find_user(&user.id).await.unwrap().is_some());
}
```

## Benchmarking with Criterion

```rust
// benches/user_benchmarks.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn create_user_benchmark(c: &mut Criterion) {
    let service = UserService::new(InMemoryRepository::new());

    c.bench_function("create_user", |b| {
        b.iter(|| {
            let user = User::new(black_box("test@example.com"));
            service.create_user(user)
        })
    });
}

criterion_group!(benches, create_user_benchmark);
criterion_main!(benches);
```

## Technology Stack

```toml
[dev-dependencies]
tokio-test = "0.4"
mockall = "0.13"
proptest = "1.5"
criterion = { version = "0.5", features = ["html_reports"] }
axum-test = "16"
wiremock = "0.6"
testcontainers = "0.21"
tempfile = "3.13"
```

## See Also

- `reference.md` - Complete patterns, CI/CD, coverage
- `../core/` - Code quality standards
