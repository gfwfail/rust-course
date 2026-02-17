# 第 44 课：HTTP 客户端 (reqwest)

> 授课时间：2026-02-18  
> 关键词：reqwest, HTTP, API, async

---

## 📦 为什么选 reqwest？

```
Rust HTTP 客户端选择：
├── hyper      → 底层库，太复杂
├── ureq       → 同步，简单，但功能少
└── reqwest ✅ → 异步，功能全，生态好
```

**reqwest** 是 Rust 最流行的 HTTP 客户端，就像 PHP 的 Guzzle、JS 的 axios。

---

## 🚀 快速开始

**Cargo.toml:**
```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

---

## 🔹 简单 GET 请求

```rust
#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    // 最简单的 GET
    let body = reqwest::get("https://httpbin.org/get")
        .await?
        .text()
        .await?;
    
    println!("{}", body);
    Ok(())
}
```

**注意两个 `.await`：**
1. 第一个等待请求完成
2. 第二个等待读取响应体

---

## 🔹 解析 JSON 响应

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct GithubUser {
    login: String,
    id: u64,
    public_repos: u32,
}

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let user: GithubUser = reqwest::get(
        "https://api.github.com/users/rust-lang"
    )
    .await?
    .json()  // 自动反序列化！
    .await?;
    
    println!("用户: {}", user.login);
    println!("仓库数: {}", user.public_repos);
    Ok(())
}
```

`.json()` 直接反序列化成你的结构体，超方便！

---

## 🔹 POST 请求 + JSON Body

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
struct CreatePost {
    title: String,
    body: String,
    user_id: u32,
}

#[derive(Debug, Deserialize)]
struct PostResponse {
    id: u64,
    title: String,
}

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let client = reqwest::Client::new();
    
    let new_post = CreatePost {
        title: "Hello Rust".to_string(),
        body: "Learning reqwest!".to_string(),
        user_id: 1,
    };
    
    let response: PostResponse = client
        .post("https://jsonplaceholder.typicode.com/posts")
        .json(&new_post)  // 自动序列化 + 设置 Content-Type
        .send()
        .await?
        .json()
        .await?;
    
    println!("Created post #{}: {}", response.id, response.title);
    Ok(())
}
```

**关键点：**
- `reqwest::Client::new()` 创建可复用的客户端
- `.json(&data)` 自动序列化并设置 `Content-Type: application/json`

---

## 🔹 设置 Headers

```rust
use reqwest::header::{AUTHORIZATION, USER_AGENT};

let client = reqwest::Client::new();

let response = client
    .get("https://api.github.com/user")
    .header(USER_AGENT, "my-rust-app/1.0")
    .header(AUTHORIZATION, "Bearer ghp_xxxx...")
    .send()
    .await?;

// 或者用字符串 header 名
let response = client
    .get("https://api.example.com/data")
    .header("X-Api-Key", "your-api-key")
    .send()
    .await?;
```

---

## 🔹 查询参数 Query String

```rust
// 方式一：直接构造 URL
let url = "https://api.example.com/search?q=rust&page=1";

// 方式二：用 .query() 方法（推荐）
let params = [
    ("q", "rust"),
    ("page", "1"),
    ("limit", "20"),
];

let response = client
    .get("https://api.example.com/search")
    .query(&params)
    .send()
    .await?;

// 方式三：用结构体
#[derive(Serialize)]
struct SearchParams {
    q: String,
    page: u32,
    limit: u32,
}

let params = SearchParams {
    q: "rust".to_string(),
    page: 1,
    limit: 20,
};

let response = client
    .get("https://api.example.com/search")
    .query(&params)
    .send()
    .await?;
```

---

## 🔹 错误处理

```rust
async fn fetch_user(id: u64) -> Result<User, Box<dyn std::error::Error>> {
    let response = reqwest::get(
        format!("https://api.example.com/users/{}", id)
    ).await?;
    
    // 检查 HTTP 状态码
    if !response.status().is_success() {
        return Err(format!(
            "HTTP Error: {} - {}",
            response.status().as_u16(),
            response.status().canonical_reason().unwrap_or("Unknown")
        ).into());
    }
    
    let user = response.json::<User>().await?;
    Ok(user)
}

// 或者用 .error_for_status()
async fn fetch_user_v2(id: u64) -> Result<User, reqwest::Error> {
    reqwest::get(format!("https://api.example.com/users/{}", id))
        .await?
        .error_for_status()?  // 4xx/5xx 自动转成 Error
        .json()
        .await
}
```

---

## 🔹 复用 Client（重要！）

```rust
// ❌ 错误：每次请求都创建新 Client
async fn bad_example() {
    for i in 0..100 {
        let client = reqwest::Client::new();  // 每次都新建连接池！
        client.get("...").send().await;
    }
}

// ✅ 正确：复用 Client
async fn good_example() {
    let client = reqwest::Client::new();  // 一次创建
    
    for i in 0..100 {
        client.get("...").send().await;  // 复用连接池
    }
}
```

**为什么复用？**
- Client 内部有连接池
- 复用可以 keep-alive，避免重复握手
- 性能差距巨大！

**全局 Client 模式：**
```rust
use once_cell::sync::Lazy;

static HTTP_CLIENT: Lazy<reqwest::Client> = Lazy::new(|| {
    reqwest::Client::builder()
        .timeout(std::time::Duration::from_secs(30))
        .build()
        .expect("Failed to create HTTP client")
});

async fn call_api() -> Result<String, reqwest::Error> {
    HTTP_CLIENT.get("https://api.example.com")
        .send()
        .await?
        .text()
        .await
}
```

---

## 🔹 超时设置

```rust
use std::time::Duration;

let client = reqwest::Client::builder()
    .timeout(Duration::from_secs(10))  // 整体超时
    .connect_timeout(Duration::from_secs(5))  // 连接超时
    .build()?;

// 或者针对单个请求
let response = client
    .get("https://slow-api.example.com")
    .timeout(Duration::from_secs(60))  // 覆盖默认超时
    .send()
    .await?;
```

---

## 🔹 实战：封装 API Client

```rust
use reqwest::{Client, Error};
use serde::{Deserialize, Serialize};

pub struct GithubClient {
    client: Client,
    token: String,
}

#[derive(Debug, Deserialize)]
pub struct Repo {
    pub id: u64,
    pub name: String,
    pub full_name: String,
    pub stargazers_count: u32,
}

#[derive(Debug, Deserialize)]
pub struct Issue {
    pub id: u64,
    pub number: u32,
    pub title: String,
}

impl GithubClient {
    pub fn new(token: &str) -> Self {
        Self {
            client: Client::new(),
            token: token.to_string(),
        }
    }
    
    pub async fn get_repos(&self, user: &str) -> Result<Vec<Repo>, Error> {
        self.client
            .get(format!("https://api.github.com/users/{}/repos", user))
            .header("Authorization", format!("Bearer {}", self.token))
            .header("User-Agent", "rust-github-client")
            .send()
            .await?
            .error_for_status()?
            .json()
            .await
    }
    
    pub async fn create_issue(
        &self,
        owner: &str,
        repo: &str,
        title: &str,
        body: &str,
    ) -> Result<Issue, Error> {
        #[derive(Serialize)]
        struct CreateIssue<'a> {
            title: &'a str,
            body: &'a str,
        }
        
        self.client
            .post(format!(
                "https://api.github.com/repos/{}/{}/issues",
                owner, repo
            ))
            .header("Authorization", format!("Bearer {}", self.token))
            .header("User-Agent", "rust-github-client")
            .json(&CreateIssue { title, body })
            .send()
            .await?
            .error_for_status()?
            .json()
            .await
    }
}

// 使用示例
async fn example() -> Result<(), Error> {
    let github = GithubClient::new("ghp_xxxx");
    let repos = github.get_repos("rust-lang").await?;
    
    for repo in repos {
        println!("{}: {} stars", repo.name, repo.stargazers_count);
    }
    Ok(())
}
```

---

## 📝 课后小结

| 方法 | 用途 |
|------|------|
| `.get()/.post()` | HTTP 方法 |
| `.json(&data)` | 发送 JSON body |
| `.query(&params)` | 添加查询参数 |
| `.header()` | 设置请求头 |
| `.send().await` | 发送请求 |
| `.text().await` | 获取文本响应 |
| `.json().await` | 解析 JSON 响应 |
| `.error_for_status()` | 4xx/5xx 转 Error |

**Laravel/Guzzle 对比：**
```php
// PHP Guzzle
$response = $client->request('POST', '/api', [
    'json' => ['name' => 'test']
]);
$data = json_decode($response->getBody());
```
```rust
// Rust reqwest
let data: MyData = client
    .post("/api")
    .json(&CreateData { name: "test" })
    .send().await?
    .json().await?;
```

**核心记忆：**
- 复用 `Client`，别每次 new
- 两个 `.await`（发送 + 读取）
- `.json()` 配合 Serde 自动序列化

---

## 🔗 相关资源

- [reqwest crate](https://crates.io/crates/reqwest)
- [reqwest 文档](https://docs.rs/reqwest)

---

*下节课预告：日期时间处理 (chrono)*
