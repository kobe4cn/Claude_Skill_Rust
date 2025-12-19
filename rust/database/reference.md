# Rust Database Reference

## Complete Entity Example

```rust
use sqlx::{FromRow, PgPool};
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, FromRow, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct User {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub is_active: bool,
}

#[derive(Debug, Clone, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CreateUserRequest {
    pub username: String,
    pub email: String,
}

#[derive(Debug, Clone, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct UpdateUserRequest {
    pub username: Option<String>,
    pub email: Option<String>,
    pub is_active: Option<bool>,
}
```

## Complete Repository Implementation

```rust
use async_trait::async_trait;

#[async_trait]
pub trait UserRepository {
    async fn create(&self, request: CreateUserRequest) -> Result<User, sqlx::Error>;
    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>, sqlx::Error>;
    async fn find_by_email(&self, email: &str) -> Result<Option<User>, sqlx::Error>;
    async fn update(&self, id: Uuid, request: UpdateUserRequest) -> Result<Option<User>, sqlx::Error>;
    async fn delete(&self, id: Uuid) -> Result<bool, sqlx::Error>;
    async fn list(&self, limit: i64, offset: i64) -> Result<Vec<User>, sqlx::Error>;
}

pub struct PostgresUserRepository {
    pool: PgPool,
}

impl PostgresUserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl UserRepository for PostgresUserRepository {
    async fn create(&self, request: CreateUserRequest) -> Result<User, sqlx::Error> {
        let id = Uuid::new_v4();
        let now = Utc::now();

        sqlx::query_as::<_, User>(
            "INSERT INTO users (id, username, email, created_at, updated_at, is_active)
             VALUES ($1, $2, $3, $4, $5, $6)
             RETURNING id, username, email, created_at, updated_at, is_active"
        )
        .bind(id)
        .bind(&request.username)
        .bind(&request.email)
        .bind(now)
        .bind(now)
        .bind(true)
        .fetch_one(&self.pool)
        .await
    }

    async fn find_by_id(&self, id: Uuid) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as::<_, User>(
            "SELECT id, username, email, created_at, updated_at, is_active
             FROM users WHERE id = $1"
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await
    }

    async fn update(&self, id: Uuid, request: UpdateUserRequest) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as::<_, User>(
            "UPDATE users
             SET username = COALESCE($2, username),
                 email = COALESCE($3, email),
                 is_active = COALESCE($4, is_active),
                 updated_at = $5
             WHERE id = $1
             RETURNING id, username, email, created_at, updated_at, is_active"
        )
        .bind(id)
        .bind(request.username)
        .bind(request.email)
        .bind(request.is_active)
        .bind(Utc::now())
        .fetch_optional(&self.pool)
        .await
    }

    async fn delete(&self, id: Uuid) -> Result<bool, sqlx::Error> {
        let result = sqlx::query("DELETE FROM users WHERE id = $1")
            .bind(id)
            .execute(&self.pool)
            .await?;

        Ok(result.rows_affected() > 0)
    }

    async fn list(&self, limit: i64, offset: i64) -> Result<Vec<User>, sqlx::Error> {
        sqlx::query_as::<_, User>(
            "SELECT id, username, email, created_at, updated_at, is_active
             FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2"
        )
        .bind(limit)
        .bind(offset)
        .fetch_all(&self.pool)
        .await
    }
}
```

## Migration Patterns

```sql
-- migrations/20240501000001_create_users_table.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_active BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);

CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

## Connection Pool Configuration

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

pub async fn create_pool(database_url: &str) -> Result<PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(20)
        .min_connections(5)
        .acquire_timeout(Duration::from_secs(30))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .connect(database_url)
        .await
}

pub async fn setup_database(config: &DatabaseConfig) -> Result<PgPool, Box<dyn std::error::Error>> {
    let pool = create_pool(&config.url).await?;
    sqlx::migrate!("./migrations").run(&pool).await?;
    Ok(pool)
}
```

## Testing with sqlx-db-tester

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use sqlx_db_tester::TestPg;

    async fn setup_test_db() -> PgPool {
        let tester = TestPg::new(
            "postgres://postgres:password@localhost/test".to_string(),
            std::path::Path::new("./migrations"),
        );
        let pool = tester.get_pool().await;
        sqlx::migrate!("./migrations").run(&pool).await.unwrap();
        pool
    }

    #[tokio::test]
    async fn test_create_user() {
        let pool = setup_test_db().await;
        let repo = PostgresUserRepository::new(pool);

        let request = CreateUserRequest {
            username: "testuser".to_string(),
            email: "test@example.com".to_string(),
        };

        let user = repo.create(request).await.unwrap();
        assert_eq!(user.username, "testuser");
        assert!(user.is_active);
    }

    #[tokio::test]
    async fn test_find_by_email() {
        let pool = setup_test_db().await;
        let repo = PostgresUserRepository::new(pool);

        let create_request = CreateUserRequest {
            username: "findme".to_string(),
            email: "findme@example.com".to_string(),
        };
        let created = repo.create(create_request).await.unwrap();

        let found = repo.find_by_email("findme@example.com").await.unwrap();
        assert!(found.is_some());
        assert_eq!(found.unwrap().id, created.id);
    }
}
```

## Database Checklist

```markdown
### Entity Design
- [ ] Uses SQLx (not rusqlite/tokio-postgres)
- [ ] Entities derive FromRow, Serialize, Deserialize
- [ ] All serde structs use #[serde(rename_all = "camelCase")]

### Query Patterns
- [ ] Uses sqlx::query_as instead of query! macro
- [ ] Parameterized queries (no SQL injection)
- [ ] Proper null handling with Option<T>

### Repository Pattern
- [ ] Async traits with async_trait
- [ ] CRUD operations implemented
- [ ] Proper error handling

### Connection Management
- [ ] Connection pool properly configured
- [ ] Timeouts set appropriately
- [ ] Migrations run on startup

### Testing
- [ ] Tests use sqlx-db-tester
- [ ] Tests cover CRUD operations
- [ ] Each test has isolated database
```

---
*Source: cursor-rust-rules features/database.mdc*
