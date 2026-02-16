# 第 32 课：Serde 序列化与反序列化

> 授课时间：2026-02-16 18:00 (AEDT)

---

## 什么是 Serde？

Serde 是 Rust 生态最强大的序列化框架，名字来自 **Ser**ialize + **De**serialize。

**类比其他语言：**
- PHP: `json_encode()` / `json_decode()`
- JS: `JSON.stringify()` / `JSON.parse()`
- Rust: Serde

但 Serde 比它们强大得多——它是**格式无关**的框架，支持 JSON、YAML、TOML、MessagePack、BSON 等几十种格式！

---

## 基础用法

### 添加依赖

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"  # JSON 格式支持
```

### 最简单的例子

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    name: String,
    age: u32,
    email: String,
}

fn main() {
    // 创建结构体
    let user = User {
        name: "老板".to_string(),
        age: 25,
        email: "boss@example.com".to_string(),
    };
    
    // 序列化：Rust → JSON
    let json = serde_json::to_string(&user).unwrap();
    println!("{}", json);
    // {"name":"老板","age":25,"email":"boss@example.com"}
    
    // 反序列化：JSON → Rust
    let parsed: User = serde_json::from_str(&json).unwrap();
    println!("{:?}", parsed);
}
```

---

## 字段属性：精细控制

### `#[serde(rename)]` - 重命名字段

```rust
#[derive(Serialize, Deserialize)]
struct ApiResponse {
    #[serde(rename = "userId")]  // JSON 用驼峰
    user_id: i64,                // Rust 用蛇形
    
    #[serde(rename = "createdAt")]
    created_at: String,
}
```

### `#[serde(rename_all)]` - 批量重命名

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]  // 自动转换所有字段
struct User {
    user_id: i64,      // → "userId"
    first_name: String, // → "firstName"
    created_at: String, // → "createdAt"
}
```

支持的命名风格：
- `camelCase` → userId
- `snake_case` → user_id
- `PascalCase` → UserId
- `SCREAMING_SNAKE_CASE` → USER_ID
- `kebab-case` → user-id

### `#[serde(skip)]` - 跳过字段

```rust
#[derive(Serialize, Deserialize)]
struct User {
    username: String,
    
    #[serde(skip)]  // 不序列化、不反序列化
    password_hash: String,
}
```

也可以单向跳过：
- `#[serde(skip_serializing)]` - 只跳过序列化
- `#[serde(skip_deserializing)]` - 只跳过反序列化

---

## 处理可选字段

### `#[serde(default)]` - 默认值

```rust
#[derive(Serialize, Deserialize)]
struct Config {
    host: String,
    
    #[serde(default)]  // 缺失时用 Default::default()
    port: u16,         // → 0
    
    #[serde(default = "default_timeout")]
    timeout: u64,
}

fn default_timeout() -> u64 {
    30  // 默认 30 秒
}
```

### `#[serde(skip_serializing_if)]` - 条件跳过

```rust
#[derive(Serialize, Deserialize)]
struct User {
    name: String,
    
    #[serde(skip_serializing_if = "Option::is_none")]
    nickname: Option<String>,  // None 时不输出字段
    
    #[serde(skip_serializing_if = "Vec::is_empty")]
    tags: Vec<String>,  // 空数组时不输出
}
```

### 组合使用

```rust
#[derive(Serialize, Deserialize)]
struct ApiParams {
    #[serde(default, skip_serializing_if = "Option::is_none")]
    page: Option<u32>,
    
    #[serde(default = "default_limit", skip_serializing_if = "is_default_limit")]
    limit: u32,
}

fn default_limit() -> u32 { 20 }
fn is_default_limit(v: &u32) -> bool { *v == 20 }
```

---

## Enum 序列化

### 默认格式（外部标记）

```rust
#[derive(Serialize, Deserialize)]
enum Message {
    Text(String),
    Image { url: String, width: u32 },
    Quit,
}

// Text("hello") → {"Text":"hello"}
// Image{...}   → {"Image":{"url":"...","width":100}}
// Quit         → "Quit"
```

### `#[serde(tag)]` - 内部标记

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Event {
    UserCreated { user_id: i64, name: String },
    UserDeleted { user_id: i64 },
}

// → {"type":"UserCreated","user_id":1,"name":"老板"}
```

### `#[serde(tag, content)]` - 相邻标记

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type", content = "data")]
enum Response {
    Success(User),
    Error(String),
}

// → {"type":"Success","data":{"name":"老板",...}}
```

### `#[serde(untagged)]` - 无标记

```rust
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
enum StringOrNumber {
    Str(String),
    Num(i64),
}

// "hello" → Str("hello")
// 42      → Num(42)
```

---

## 实战：对接第三方 API

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Deserialize)]
struct ApiResponse<T> {
    code: i32,
    message: String,
    data: T,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
struct UserData {
    user_id: i64,
    user_name: String,
    created_at: String,
    profile: Profile,
}

#[derive(Debug, Deserialize)]
struct Profile {
    avatar_url: Option<String>,
    bio: Option<String>,
}

async fn fetch_user() -> Result<UserData, Error> {
    let resp: ApiResponse<UserData> = client
        .get("https://api.example.com/user/123")
        .send()
        .await?
        .json()
        .await?;
    
    if resp.code != 0 {
        return Err(anyhow!("API error: {}", resp.message));
    }
    
    Ok(resp.data)
}
```

---

## 与 Axum 整合

```rust
use axum::{Json, extract::Path};
use serde::{Serialize, Deserialize};

// 请求体
#[derive(Deserialize)]
#[serde(rename_all = "camelCase")]
struct CreateUserRequest {
    user_name: String,
    email: String,
    #[serde(default)]
    is_admin: bool,
}

// 响应体
#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
struct UserResponse {
    user_id: i64,
    user_name: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    avatar_url: Option<String>,
}

// Handler
async fn create_user(
    Json(req): Json<CreateUserRequest>,
) -> Json<UserResponse> {
    println!("Creating user: {}", req.user_name);
    
    Json(UserResponse {
        user_id: 1,
        user_name: req.user_name,
        avatar_url: None,
    })
}
```

---

## 其他格式支持

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"     # JSON
serde_yaml = "0.9"   # YAML
toml = "0.8"         # TOML
```

```rust
#[derive(Serialize, Deserialize)]
struct Config {
    database_url: String,
    port: u16,
}

let config = Config {
    database_url: "postgres://...".to_string(),
    port: 8080,
};

// 同一个结构体，输出不同格式
let json = serde_json::to_string_pretty(&config)?;
let yaml = serde_yaml::to_string(&config)?;
let toml = toml::to_string(&config)?;
```

---

## 总结

| 属性 | 作用 |
|------|------|
| `#[derive(Serialize, Deserialize)]` | 启用序列化 |
| `#[serde(rename = "...")]` | 重命名单个字段 |
| `#[serde(rename_all = "camelCase")]` | 批量重命名 |
| `#[serde(skip)]` | 跳过字段 |
| `#[serde(default)]` | 设置默认值 |
| `#[serde(skip_serializing_if = "...")]` | 条件跳过 |
| `#[serde(tag = "type")]` | Enum 内部标记 |
| `#[serde(untagged)]` | Enum 无标记 |

**一句话：**
> Serde = Rust 的数据转换瑞士军刀，Web 开发必备技能！

---

*下节课预告：Tower 中间件*
