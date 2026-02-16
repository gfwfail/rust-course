# 第30课：SQLx 数据库操作

> 🕐 授课时间：2026-02-16 12:00  
> 📍 地点：Telegram 群 Rust学习小组

---

## 什么是 SQLx？

SQLx 是 Rust 最流行的**异步数据库库**，特点：

- **编译期 SQL 检查**：能检查 SQL 语法和类型
- **原生异步**：基于 Tokio/async-std
- **无 ORM**：写原生 SQL，性能透明
- **多数据库支持**：PostgreSQL / MySQL / SQLite

对比 PHP 世界：
- Laravel Eloquent = ORM，写 PHP 代码
- SQLx = 写原生 SQL，但类型安全

---

## 添加依赖

```toml
# Cargo.toml
[dependencies]
sqlx = { version = "0.8", features = [
    "runtime-tokio",   # 使用 tokio
    "postgres",        # PostgreSQL
    "chrono",          # 时间类型支持
    "uuid",            # UUID 支持
] }
tokio = { version = "1", features = ["full"] }
dotenvy = "0.15"      # 读取 .env
```

如果用 MySQL：
```toml
sqlx = { version = "0.8", features = ["runtime-tokio", "mysql"] }
```

---

## 连接数据库

```rust
use sqlx::postgres::PgPoolOptions;
use std::env;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    // 从环境变量读取
    dotenvy::dotenv().ok();
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");

    // 创建连接池
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await?;

    println!("✅ 数据库连接成功！");
    Ok(())
}
```

`.env` 文件：
```env
DATABASE_URL=postgres://user:password@localhost/mydb
```

⚠️ **连接池很重要**：Web 应用每个请求都要查数据库，复用连接能显著提升性能。

---

## 基本查询

### query! 宏 - 编译期检查

```rust
// 查询多行
let users = sqlx::query!("SELECT id, name, email FROM users")
    .fetch_all(&pool)
    .await?;

for user in users {
    println!("{}: {} ({})", user.id, user.name, user.email);
}
```

`query!` 宏会在**编译期**检查：
- SQL 语法是否正确
- 表和字段是否存在
- 返回的类型是否匹配

### 带参数查询

```rust
// $1, $2... 是占位符（PostgreSQL 风格）
let user = sqlx::query!(
    "SELECT id, name, email FROM users WHERE id = $1",
    user_id  // 绑定参数
)
.fetch_one(&pool)
.await?;

println!("找到用户: {}", user.name);
```

MySQL 用 `?` 占位符：
```rust
sqlx::query!("SELECT * FROM users WHERE id = ?", user_id)
```

### fetch 变体

```rust
// fetch_all - 获取所有行
let all_users = sqlx::query!("SELECT * FROM users")
    .fetch_all(&pool).await?;

// fetch_one - 获取一行（没有则报错）
let user = sqlx::query!("SELECT * FROM users WHERE id = $1", 1)
    .fetch_one(&pool).await?;

// fetch_optional - 获取一行（没有则 None）
let maybe_user = sqlx::query!("SELECT * FROM users WHERE id = $1", 999)
    .fetch_optional(&pool).await?;

if let Some(user) = maybe_user {
    println!("找到: {}", user.name);
} else {
    println!("用户不存在");
}
```

---

## 映射到结构体

### query_as! 宏

```rust
use sqlx::FromRow;

#[derive(Debug, FromRow)]
struct User {
    id: i64,
    name: String,
    email: String,
    created_at: chrono::DateTime<chrono::Utc>,
}

// 自动映射到 User 结构体
let users: Vec<User> = sqlx::query_as!(
    User,
    "SELECT id, name, email, created_at FROM users"
)
.fetch_all(&pool)
.await?;

for user in users {
    println!("{:?}", user);
}
```

### 字段重命名

如果数据库字段名和结构体字段名不一致：

```rust
#[derive(Debug, FromRow)]
struct User {
    id: i64,
    #[sqlx(rename = "user_name")]
    name: String,
}

// 或者在 SQL 里用 AS
let users = sqlx::query_as!(
    User,
    "SELECT id, user_name AS name FROM users"
)
.fetch_all(&pool)
.await?;
```

---

## INSERT / UPDATE / DELETE

### 插入数据

```rust
// 插入并返回生成的 ID
let result = sqlx::query!(
    "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
    "张三",
    "zhang@test.com"
)
.fetch_one(&pool)
.await?;

println!("新用户 ID: {}", result.id);
```

MySQL 版本（不支持 RETURNING）：
```rust
let result = sqlx::query!(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    "张三",
    "zhang@test.com"
)
.execute(&pool)
.await?;

println!("新用户 ID: {}", result.last_insert_id());
```

### 更新数据

```rust
let result = sqlx::query!(
    "UPDATE users SET name = $1 WHERE id = $2",
    "李四",
    user_id
)
.execute(&pool)
.await?;

println!("影响 {} 行", result.rows_affected());
```

### 删除数据

```rust
let result = sqlx::query!(
    "DELETE FROM users WHERE id = $1",
    user_id
)
.execute(&pool)
.await?;

if result.rows_affected() > 0 {
    println!("删除成功");
} else {
    println!("用户不存在");
}
```

---

## 事务处理

事务确保多个操作要么全部成功，要么全部回滚：

```rust
// 开启事务
let mut tx = pool.begin().await?;

// 扣款
sqlx::query!(
    "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
    100,
    from_account_id
)
.execute(&mut *tx)  // 注意：传 &mut *tx
.await?;

// 加款
sqlx::query!(
    "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
    100,
    to_account_id
)
.execute(&mut *tx)
.await?;

// 提交事务（如果不调用 commit，事务会自动回滚）
tx.commit().await?;

println!("转账成功！");
```

### 自动回滚

```rust
async fn transfer(pool: &PgPool) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;

    sqlx::query!("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
        .execute(&mut *tx).await?;

    // 如果这里出错，tx 会在函数结束时 drop，自动回滚
    sqlx::query!("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
        .execute(&mut *tx).await?;

    tx.commit().await?;  // 成功才提交
    Ok(())
}
```

---

## 和 Axum 集成

把 SQLx 连接池注入到 Axum 的 State 中：

```rust
use axum::{
    Router, Json,
    routing::{get, post},
    extract::{Path, State},
    http::StatusCode,
};
use sqlx::postgres::PgPoolOptions;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

type DbPool = sqlx::PgPool;

// 获取所有用户
async fn list_users(
    State(pool): State<DbPool>,
) -> Result<Json<Vec<User>>, StatusCode> {
    let users = sqlx::query_as!(User, "SELECT id, name, email FROM users")
        .fetch_all(&pool)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(users))
}

// 获取单个用户
async fn get_user(
    State(pool): State<DbPool>,
    Path(id): Path<i64>,
) -> Result<Json<User>, StatusCode> {
    let user = sqlx::query_as!(
        User,
        "SELECT id, name, email FROM users WHERE id = $1",
        id
    )
    .fetch_optional(&pool)
    .await
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
    .ok_or(StatusCode::NOT_FOUND)?;

    Ok(Json(user))
}

// 创建用户
async fn create_user(
    State(pool): State<DbPool>,
    Json(payload): Json<CreateUser>,
) -> Result<(StatusCode, Json<User>), StatusCode> {
    let user = sqlx::query_as!(
        User,
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, name, email",
        payload.name,
        payload.email
    )
    .fetch_one(&pool)
    .await
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok((StatusCode::CREATED, Json(user)))
}

#[tokio::main]
async fn main() {
    dotenvy::dotenv().ok();
    let database_url = std::env::var("DATABASE_URL").unwrap();

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("连接数据库失败");

    let app = Router::new()
        .route("/users", get(list_users).post(create_user))
        .route("/users/:id", get(get_user))
        .with_state(pool);  // 注入连接池

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await.unwrap();
    
    println!("🚀 Server at http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

## 数据库迁移 (Migrations)

SQLx 有内置的迁移工具，类似 Laravel 的 `php artisan migrate`。

### 安装 CLI

```bash
cargo install sqlx-cli --no-default-features --features postgres
```

### 创建迁移

```bash
sqlx migrate add create_users_table
```

这会创建 `migrations/20260216_create_users_table.sql`：

```sql
-- migrations/20260216120000_create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 运行迁移

```bash
# 运行所有迁移
sqlx migrate run

# 查看迁移状态
sqlx migrate info

# 回滚最后一次迁移
sqlx migrate revert
```

### 在代码中运行迁移

```rust
sqlx::migrate!("./migrations")
    .run(&pool)
    .await?;
```

对比 Laravel：
```bash
php artisan make:migration create_users_table
php artisan migrate
php artisan migrate:rollback
```

---

## 💡 对比 Laravel

| 概念 | Laravel | SQLx |
|------|---------|------|
| ORM 查询 | `User::all()` | `query_as!` + SQL |
| 条件查询 | `->where('id', 1)` | `WHERE id = $1` |
| 插入 | `User::create([...])` | `INSERT ... RETURNING` |
| 事务 | `DB::transaction()` | `pool.begin()` |
| 迁移 | `artisan migrate` | `sqlx migrate run` |
| 连接池 | 自动管理 | `PgPoolOptions` |

**核心区别**：
- Laravel Eloquent 是 ORM，抽象掉 SQL
- SQLx 是查询构建器，直接写 SQL 但类型安全

---

## 🧠 本课小结

1. **SQLx** 是 Rust 最流行的异步数据库库，编译期检查 SQL
2. **连接池** 用 `PgPoolOptions::new().connect()` 创建
3. **query!** 和 **query_as!** 宏执行查询，自动类型推断
4. **fetch_all / fetch_one / fetch_optional** 处理不同场景
5. **事务** 用 `pool.begin()` 开启，`tx.commit()` 提交
6. **与 Axum 集成**：把 Pool 放到 `State` 中注入

---

## 📖 推荐资源

- [SQLx 官方文档](https://docs.rs/sqlx)
- [SQLx GitHub](https://github.com/launchbadge/sqlx)

---

*下节课：错误处理最佳实践（anyhow / thiserror）*
