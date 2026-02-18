# 第 46 课：UUID 生成与使用 (uuid crate)

> 日期：2026-02-18  
> 主题：uuid crate 的使用

---

## 为什么需要 UUID？

在 Web 开发中，我们经常需要生成唯一标识符：
- 数据库主键（替代自增 ID）
- API 请求追踪 ID
- 会话 Token
- 文件名去重

UUID（Universally Unique Identifier）是一种 128 位的唯一标识符，格式如：
```
550e8400-e29b-41d4-a716-446655440000
```

---

## 添加依赖

```toml
# Cargo.toml
[dependencies]
uuid = { version = "1.16", features = ["v4", "v7", "serde"] }
```

常用 features：
- `v4` - 随机生成（最常用）
- `v7` - 时间戳 + 随机（可排序，新项目推荐）
- `serde` - 序列化支持
- `js` - WASM 支持

---

## 基础用法

```rust
use uuid::Uuid;

fn main() {
    // 生成 v4 UUID（随机）
    let id = Uuid::new_v4();
    println!("UUID v4: {}", id);
    // 输出: UUID v4: 67e55044-10b1-426f-9247-bb680e5fe0c8

    // 生成 v7 UUID（时间戳 + 随机，可排序）
    let id_v7 = Uuid::now_v7();
    println!("UUID v7: {}", id_v7);
    // 输出: UUID v7: 0192d4a1-8c3b-7def-8a12-f4b3c5e6d7a8

    // 解析字符串
    let parsed = Uuid::parse_str("550e8400-e29b-41d4-a716-446655440000").unwrap();
    println!("Parsed: {}", parsed);

    // 转换为不同格式
    println!("Hyphenated: {}", id.hyphenated()); // 带连字符（默认）
    println!("Simple: {}", id.simple());         // 无连字符
    println!("URN: {}", id.urn());               // urn:uuid:...
}
```

---

## v4 vs v7：怎么选？

| 特性 | v4 | v7 |
|-----|----|----|
| 生成方式 | 纯随机 | 时间戳 + 随机 |
| 可排序 | ❌ | ✅ 按时间排序 |
| 数据库性能 | 一般 | 更好（B-tree 友好）|
| 隐私 | ✅ 不含时间信息 | ⚠️ 可推断创建时间 |
| 推荐场景 | Token、临时 ID | 数据库主键、日志 ID |

**2024+ 新项目建议用 v7**，数据库索引性能更好。

---

## 实战：与 SQLx 结合

```rust
use sqlx::FromRow;
use uuid::Uuid;
use serde::{Deserialize, Serialize};

#[derive(Debug, FromRow, Serialize, Deserialize)]
struct User {
    id: Uuid,          // 直接用 Uuid 类型
    username: String,
    email: String,
}

async fn create_user(pool: &sqlx::PgPool, username: &str, email: &str) -> Result<User, sqlx::Error> {
    let user = sqlx::query_as::<_, User>(
        r#"
        INSERT INTO users (id, username, email)
        VALUES ($1, $2, $3)
        RETURNING *
        "#
    )
    .bind(Uuid::now_v7())  // 使用 v7，方便按创建时间排序
    .bind(username)
    .bind(email)
    .fetch_one(pool)
    .await?;

    Ok(user)
}
```

---

## 实战：与 Axum 结合

```rust
use axum::{extract::Path, routing::get, Json, Router};
use uuid::Uuid;
use serde::Serialize;

#[derive(Serialize)]
struct Response {
    id: Uuid,
    message: String,
}

// 从路径解析 UUID
async fn get_user(Path(user_id): Path<Uuid>) -> Json<Response> {
    Json(Response {
        id: user_id,
        message: format!("Found user: {}", user_id),
    })
}

// 生成新资源
async fn create_resource() -> Json<Response> {
    Json(Response {
        id: Uuid::now_v7(),
        message: "Resource created".to_string(),
    })
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/users/{user_id}", get(get_user))
        .route("/resources", axum::routing::post(create_resource));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

## 常见操作

```rust
use uuid::Uuid;

fn main() {
    let id = Uuid::new_v4();

    // 检查是否为空（nil UUID）
    println!("Is nil: {}", id.is_nil());        // false
    println!("Nil UUID: {}", Uuid::nil());      // 00000000-0000-0000-0000-000000000000

    // 获取版本
    println!("Version: {:?}", id.get_version()); // Some(Random)

    // 获取底层字节
    let bytes: &[u8; 16] = id.as_bytes();
    println!("Bytes: {:?}", bytes);

    // 从字节创建
    let from_bytes = Uuid::from_bytes(*bytes);
    assert_eq!(id, from_bytes);

    // 比较
    let id2 = Uuid::new_v4();
    println!("Same? {}", id == id2); // false（几乎不可能相同）
}
```

---

## 最佳实践

1. **数据库主键用 v7**，索引性能好，可按时间排序
2. **Token/临时 ID 用 v4**，不泄露时间信息
3. **启用 serde feature**，方便 JSON 序列化
4. **不要手动拼接 UUID 字符串**，用 `Uuid::parse_str()`

---

## 课后练习

1. 生成 10 个 v7 UUID，观察它们的排序顺序
2. 写一个函数，接受 `&str` 参数，验证是否是合法 UUID
3. 在 Axum 路由中使用 `Uuid` 作为路径参数

---

📚 文档：https://docs.rs/uuid
