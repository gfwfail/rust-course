# 第31课：错误处理最佳实践（anyhow / thiserror）

> 🕐 授课时间：2026-02-16 15:00  
> 📍 地点：Telegram 群 Rust学习小组

---

## 先说结论

```
应用程序（bin）→ 用 anyhow
库（lib）      → 用 thiserror
```

这是 Rust 社区的最佳实践。

---

## 为什么需要这两个库？

标准库的 `Result<T, E>` 有个问题：**E 必须是具体类型**。

```rust
// 这函数可能返回 IO 错误，也可能返回解析错误
fn read_config() -> Result<Config, ???> {
    let content = std::fs::read_to_string("config.json")?; // io::Error
    let config: Config = serde_json::from_str(&content)?;  // serde_json::Error
    Ok(config)
}
```

两种错误类型不一样，怎么写返回类型？

### 方案一：Box<dyn Error>（能用但不爽）

```rust
use std::error::Error;

fn read_config() -> Result<Config, Box<dyn Error>> {
    let content = std::fs::read_to_string("config.json")?;
    let config = serde_json::from_str(&content)?;
    Ok(config)
}
```

问题：
- 丢失了具体错误类型
- 调用者无法 match 具体错误
- 性能有堆分配开销

---

## anyhow - 应用级错误处理

`anyhow` 提供了 `anyhow::Error` 类型，可以包装任何实现了 `std::error::Error` 的类型。

### 添加依赖

```toml
[dependencies]
anyhow = "1.0"
```

### 基本用法

```rust
use anyhow::{Result, Context};

// Result<T> 是 Result<T, anyhow::Error> 的别名
fn read_config() -> Result<Config> {
    let content = std::fs::read_to_string("config.json")?;
    let config = serde_json::from_str(&content)?;
    Ok(config)
}

fn main() -> Result<()> {
    let config = read_config()?;
    println!("配置加载成功: {:?}", config);
    Ok(())
}
```

### Context - 添加上下文信息

这是 anyhow 最强的功能：给错误添加上下文，方便调试。

```rust
use anyhow::{Result, Context};

fn read_user_config(user_id: i64) -> Result<Config> {
    let path = format!("/home/{}/config.json", user_id);
    
    // 如果文件不存在，错误信息会是：
    // "读取用户配置失败 (user_id=42)"
    // Caused by: No such file or directory
    let content = std::fs::read_to_string(&path)
        .with_context(|| format!("读取用户配置失败 (user_id={})", user_id))?;
    
    let config = serde_json::from_str(&content)
        .context("解析配置 JSON 失败")?;
    
    Ok(config)
}
```

### anyhow! 和 bail! 宏

```rust
use anyhow::{anyhow, bail, Result, ensure};

fn validate_age(age: i32) -> Result<()> {
    // bail! = return Err(anyhow!(...))
    if age < 0 {
        bail!("年龄不能为负数: {}", age);
    }
    
    // ensure! = if !condition { bail!(...) }
    ensure!(age <= 150, "年龄不合理: {}", age);
    
    Ok(())
}

fn get_user(id: Option<i64>) -> Result<User> {
    // anyhow! 创建一个错误
    let id = id.ok_or_else(|| anyhow!("用户 ID 不能为空"))?;
    // ...
}
```

**速记：**
- `anyhow!("msg")` - 创建错误
- `bail!("msg")` - 直接返回错误（return Err）
- `ensure!(cond, "msg")` - 断言，失败则返回错误

---

## thiserror - 库级错误定义

如果你在写一个**库**给别人用，需要定义清晰的错误类型，让用户可以 match 处理。

### 添加依赖

```toml
[dependencies]
thiserror = "2.0"
```

### 基本用法

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("数据库连接失败: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("配置文件不存在: {path}")]
    ConfigNotFound { path: String },
    
    #[error("用户 {user_id} 没有权限访问 {resource}")]
    Unauthorized { user_id: i64, resource: String },
    
    #[error("未知错误")]
    Unknown,
}
```

**`#[error("...")]`** - 定义 Display 输出
**`#[from]`** - 自动实现 From trait，支持 `?` 转换

### 使用自定义错误

```rust
#[derive(Error, Debug)]
pub enum UserError {
    #[error("用户不存在: {0}")]
    NotFound(i64),
    
    #[error("邮箱已被使用: {0}")]
    EmailTaken(String),
    
    #[error("数据库错误")]
    Database(#[from] sqlx::Error),
}

fn get_user(pool: &PgPool, id: i64) -> Result<User, UserError> {
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_optional(pool)
        .await?;  // sqlx::Error 自动转为 UserError::Database
    
    user.ok_or(UserError::NotFound(id))
}

// 调用者可以 match 处理
fn main() {
    match get_user(&pool, 42).await {
        Ok(user) => println!("找到用户: {}", user.name),
        Err(UserError::NotFound(id)) => println!("用户 {} 不存在", id),
        Err(UserError::EmailTaken(email)) => println!("邮箱冲突: {}", email),
        Err(UserError::Database(e)) => eprintln!("数据库炸了: {}", e),
    }
}
```

### #[source] - 保留错误链

```rust
#[derive(Error, Debug)]
pub enum ConfigError {
    // #[from] 自动实现 From + source
    #[error("读取配置文件失败")]
    Io(#[from] std::io::Error),
    
    // #[source] 只标记为 source，不自动实现 From
    #[error("解析配置失败: {path}")]
    Parse {
        path: String,
        #[source]
        source: serde_json::Error,
    },
}
```

**`#[from]`** = 自动实现 From + 标记 source
**`#[source]`** = 只标记为 source（需要手动转换）

---

## anyhow + thiserror 配合使用

实际项目中，两者经常一起用：

```
your_app/
├── src/
│   ├── main.rs          # 用 anyhow
│   └── lib/
│       ├── user.rs      # 用 thiserror
│       └── config.rs    # 用 thiserror
```

### 库代码（thiserror）

```rust
// src/lib/user.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum UserError {
    #[error("用户不存在")]
    NotFound,
    #[error("邮箱格式错误")]
    InvalidEmail,
    #[error("数据库错误")]
    Database(#[from] sqlx::Error),
}

pub fn create_user(email: &str) -> Result<User, UserError> {
    if !email.contains('@') {
        return Err(UserError::InvalidEmail);
    }
    // ...
}
```

### 应用代码（anyhow）

```rust
// src/main.rs
use anyhow::{Result, Context};
use mylib::user::{create_user, UserError};

fn main() -> Result<()> {
    // 方式1: 直接传播，加上下文
    let user = create_user("test@example.com")
        .context("创建用户失败")?;
    
    // 方式2: 需要特殊处理某些错误
    match create_user("invalid-email") {
        Ok(user) => println!("创建成功"),
        Err(UserError::InvalidEmail) => {
            println!("请输入正确的邮箱格式");
        }
        Err(e) => return Err(e.into()),
    }
    
    Ok(())
}
```

---

## Axum 中的错误处理

### 方式1：直接返回 StatusCode

```rust
async fn get_user(
    Path(id): Path<i64>,
    State(pool): State<PgPool>,
) -> Result<Json<User>, StatusCode> {
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_optional(&pool)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
        .ok_or(StatusCode::NOT_FOUND)?;
    
    Ok(Json(user))
}
```

问题：丢失了错误信息。

### 方式2：自定义 AppError（推荐）

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("资源不存在")]
    NotFound,
    
    #[error("未授权")]
    Unauthorized,
    
    #[error("参数错误: {0}")]
    BadRequest(String),
    
    #[error("内部错误")]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound => (StatusCode::NOT_FOUND, self.to_string()),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, self.to_string()),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            AppError::Internal(e) => {
                tracing::error!("内部错误: {:?}", e);
                (StatusCode::INTERNAL_SERVER_ERROR, "服务器内部错误".to_string())
            }
        };
        
        let body = Json(json!({ "error": message }));
        (status, body).into_response()
    }
}
```

### 在 Handler 中使用

```rust
use anyhow::Context;

async fn get_user(
    Path(id): Path<i64>,
    State(pool): State<PgPool>,
) -> Result<Json<User>, AppError> {
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_optional(&pool)
        .await
        .context("查询用户失败")?
        .ok_or(AppError::NotFound)?;
    
    Ok(Json(user))
}
```

---

## 💡 对比 Laravel

| 概念 | Laravel | Rust |
|------|---------|------|
| 抛出异常 | `throw new Exception()` | `return Err(...)` |
| 捕获异常 | `try/catch` | `match` 或 `?` |
| 自定义异常 | `class MyException extends Exception` | `#[derive(Error)]` |
| 异常处理器 | `Handler::render()` | `impl IntoResponse` |
| 上下文信息 | `$e->getMessage()` | `.context()` |

**核心区别：**
- PHP：异常会中断控制流，catch 不是强制的
- Rust：`Result` 是类型，必须显式处理

---

## 🧠 本课小结

1. **anyhow** - 应用级错误处理
   - `Result<T>` 替代 `Result<T, E>`
   - `.context()` 添加上下文
   - `bail!` / `ensure!` 快速返回错误

2. **thiserror** - 库级错误定义
   - `#[derive(Error)]` 自动实现 Error trait
   - `#[error("...")]` 定义错误信息
   - `#[from]` / `#[source]` 处理错误链

3. **最佳实践**
   - 库用 thiserror 暴露结构化错误
   - 应用用 anyhow 简化传播
   - Axum 实现 `IntoResponse` 统一 API 错误格式

---

## 📖 推荐资源

- [anyhow 文档](https://docs.rs/anyhow)
- [thiserror 文档](https://docs.rs/thiserror)
- [Rust Error Handling](https://nick.groenen.me/posts/rust-error-handling/)

---

*下节课：Tower 中间件与层 - Axum 的底层魔法*
