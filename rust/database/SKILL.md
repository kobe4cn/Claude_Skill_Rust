---
description: >-
  Use when working with databases in Rust. Provides SQLx patterns including
  repository pattern, entity definitions, migrations, and testing with sqlx-db-tester.
  Triggers: SQLx, database, PostgreSQL, SQLite, repository, SQL, migrations.
---

# Rust Database Patterns (SQLx)

## Quick Reference

This skill provides guidance for database access in Rust using SQLx:
- Entity definitions with FromRow
- Repository pattern with async traits
- Query patterns with query_as
- Testing with sqlx-db-tester
- Migration management

## Key Requirements

### SQLx as Standard
- **Always use SQLx** - avoid rusqlite, tokio-postgres
- Use `sqlx::query_as` instead of `sqlx::query!` macro
- All entities must derive `FromRow`, `Serialize`, `Deserialize`
- All serde structs use `#[serde(rename_all = "camelCase")]`

## Technology Stack

```toml
[dependencies]
sqlx = { version = "0.8", features = [
    "chrono", "postgres", "runtime-tokio-rustls", "sqlite", "uuid"
] }

[dev-dependencies]
sqlx-db-tester = "0.5"
```

## Entity Definitions

```rust
#[derive(Debug, Clone, sqlx::FromRow, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct User {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub is_active: bool,
}
```

## Query Patterns

```rust
// Use query_as instead of query! macro
impl User {
    pub async fn find_by_id(
        pool: &PgPool,
        id: Uuid
    ) -> Result<Option<Self>, sqlx::Error> {
        sqlx::query_as::<_, User>(
            "SELECT id, username, email, created_at, updated_at, is_active
             FROM users WHERE id = $1"
        )
        .bind(id)
        .fetch_optional(pool)
        .await
    }
}
```

## Repository Pattern

```rust
#[async_trait]
pub trait UserRepository {
    async fn create(&self, request: CreateUserRequest) -> Result<User, sqlx::Error>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>, sqlx::Error>;
    async fn update(&self, id: Uuid, request: UpdateUserRequest) -> Result<Option<User>, sqlx::Error>;
    async fn delete(&self, id: Uuid) -> Result<bool, sqlx::Error>;
}

pub struct PostgresUserRepository {
    pool: PgPool,
}

#[async_trait]
impl UserRepository for PostgresUserRepository {
    async fn create(&self, request: CreateUserRequest) -> Result<User, sqlx::Error> {
        sqlx::query_as::<_, User>(
            "INSERT INTO users (id, username, email, created_at, updated_at, is_active)
             VALUES ($1, $2, $3, $4, $5, $6)
             RETURNING *"
        )
        .bind(Uuid::new_v4())
        .bind(&request.username)
        .bind(&request.email)
        .bind(Utc::now())
        .bind(Utc::now())
        .bind(true)
        .fetch_one(&self.pool)
        .await
    }
}
```

## Testing with sqlx-db-tester

```rust
#[cfg(test)]
mod tests {
    use sqlx_db_tester::TestPg;

    async fn setup_test_db() -> PgPool {
        let tester = TestPg::new(
            "postgres://postgres:password@localhost/test".to_string(),
            std::path::Path::new("./migrations"),
        );
        tester.get_pool().await
    }

    #[tokio::test]
    async fn test_create_user() {
        let pool = setup_test_db().await;
        let repo = PostgresUserRepository::new(pool);

        let user = repo.create(CreateUserRequest {
            username: "testuser".to_string(),
            email: "test@example.com".to_string(),
        }).await.unwrap();

        assert_eq!(user.username, "testuser");
    }
}
```

## Anti-Patterns to Avoid

- Using rusqlite or tokio-postgres directly
- Using sqlx::query! macro (compile-time dependencies)
- Entities without FromRow derive
- Missing serde camelCase configuration
- Synchronous database operations

## See Also

- `reference.md` - Complete patterns, migrations, and connection management
