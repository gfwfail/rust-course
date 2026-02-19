# 第 56 课：限流 Rate Limiting

## 为什么需要限流？

限流是保护 API 的重要手段：
- **防止滥用**：阻止恶意用户刷接口
- **保护资源**：避免服务器过载
- **公平分配**：确保所有用户都能获得服务
- **成本控制**：避免第三方 API 调用费用暴涨

PHP/Laravel 里有 `throttle` 中间件，Rust 里我们用 `tower` 生态。

---

## 常用限流 Crate

```toml
[dependencies]
tower = "0.5"
tower_governor = "0.4"       # 基于 Governor 的 Tower 中间件
governor = "0.7"             # 底层限流库
axum = "0.8"
```

`governor` 是 Rust 最流行的限流库，`tower_governor` 把它封装成了 Tower 中间件。

---

## 基本概念：令牌桶算法

限流最常用的是**令牌桶 (Token Bucket)** 算法：

```
┌─────────────────┐
│  Token Bucket   │
│  ┌───┬───┬───┐  │
│  │ 🪙│ 🪙│ 🪙│  │  ← 桶容量 (burst)
│  └───┴───┴───┘  │
│        ↑        │
│    每秒补充      │  ← 填充速率 (rate)
└─────────────────┘
```

- **桶容量 (burst)**：最多存多少令牌
- **填充速率 (rate)**：每秒补充多少令牌
- 每次请求消耗一个令牌
- 令牌用完 = 限流

---

## 实战：全局限流

最简单的全局限流（所有用户共享一个限制）：

```rust
use axum::{routing::get, Router};
use std::time::Duration;
use tower_governor::{
    governor::GovernorConfigBuilder,
    GovernorLayer,
};

#[tokio::main]
async fn main() {
    // 配置：每秒 10 个请求，最多突发 20 个
    let governor_conf = GovernorConfigBuilder::default()
        .per_second(10)          // 每秒补充 10 个令牌
        .burst_size(20)          // 桶容量 20
        .finish()
        .unwrap();

    let app = Router::new()
        .route("/api/data", get(get_data))
        .layer(GovernorLayer {
            config: governor_conf,
        });

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn get_data() -> &'static str {
    "Hello, World!"
}
```

超过限制时，自动返回 `429 Too Many Requests`。

---

## 按 IP 限流（更常用）

实际应用中，我们通常按用户 IP 限流：

```rust
use axum::{
    routing::get,
    Router,
    extract::ConnectInfo,
};
use std::net::SocketAddr;
use tower_governor::{
    governor::GovernorConfigBuilder,
    key_extractor::SmartIpKeyExtractor,
    GovernorLayer,
};

#[tokio::main]
async fn main() {
    // 按 IP 限流：每个 IP 每秒 5 个请求
    let governor_conf = GovernorConfigBuilder::default()
        .per_second(5)
        .burst_size(10)
        .key_extractor(SmartIpKeyExtractor)  // 🔑 按 IP 区分
        .finish()
        .unwrap();

    let app = Router::new()
        .route("/api/data", get(get_data))
        .layer(GovernorLayer {
            config: governor_conf,
        });

    // 需要 into_make_service_with_connect_info 获取客户端 IP
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await
    .unwrap();
}

async fn get_data() -> &'static str {
    "Data for you!"
}
```

`SmartIpKeyExtractor` 会自动处理：
- 直连 IP
- `X-Forwarded-For` 头（反向代理场景）
- `X-Real-IP` 头

---

## 自定义 Key Extractor

按用户 ID 限流（需要登录的 API）：

```rust
use axum::{
    extract::Request,
    http::StatusCode,
};
use tower_governor::key_extractor::KeyExtractor;

#[derive(Clone)]
struct UserIdKeyExtractor;

impl KeyExtractor for UserIdKeyExtractor {
    type Key = String;

    fn extract<T>(&self, req: &Request<T>) -> Result<Self::Key, GovernorError> {
        // 从请求头获取用户 ID（实际中可能从 JWT 解析）
        req.headers()
            .get("X-User-Id")
            .and_then(|v| v.to_str().ok())
            .map(|s| s.to_string())
            .ok_or(GovernorError::UnableToExtractKey)
    }

    // 限流时的响应
    fn response_error() -> StatusCode {
        StatusCode::TOO_MANY_REQUESTS
    }

    fn name(&self) -> &'static str {
        "user_id"
    }
}

// 使用
let governor_conf = GovernorConfigBuilder::default()
    .per_second(100)
    .burst_size(200)
    .key_extractor(UserIdKeyExtractor)
    .finish()
    .unwrap();
```

---

## 不同时间窗口

```rust
// 每秒 10 个
GovernorConfigBuilder::default()
    .per_second(10)
    .burst_size(10)

// 每分钟 60 个
GovernorConfigBuilder::default()
    .period(Duration::from_secs(60))
    .per_period(60)
    .burst_size(60)

// 每小时 1000 个（适合付费 API）
GovernorConfigBuilder::default()
    .period(Duration::from_secs(3600))
    .per_period(1000)
    .burst_size(100)  // 允许短期突发 100 个
```

---

## 不同路由不同限制

```rust
use axum::{routing::{get, post}, Router};
use tower_governor::{governor::GovernorConfigBuilder, GovernorLayer};

#[tokio::main]
async fn main() {
    // 登录接口：严格限制（防暴力破解）
    let login_limit = GovernorConfigBuilder::default()
        .per_second(1)
        .burst_size(3)  // 每秒 1 次，最多连续 3 次
        .key_extractor(SmartIpKeyExtractor)
        .finish()
        .unwrap();

    // 普通 API：宽松限制
    let api_limit = GovernorConfigBuilder::default()
        .per_second(50)
        .burst_size(100)
        .key_extractor(SmartIpKeyExtractor)
        .finish()
        .unwrap();

    let app = Router::new()
        // 登录路由单独限流
        .route("/auth/login", post(login))
            .layer(GovernorLayer { config: login_limit })
        // 其他 API 统一限流
        .nest(
            "/api",
            Router::new()
                .route("/users", get(list_users))
                .route("/posts", get(list_posts))
                .layer(GovernorLayer { config: api_limit }),
        );

    // ...
}
```

---

## 返回限流信息（RateLimit Headers）

告诉客户端还剩多少配额：

```rust
use tower_governor::{
    governor::GovernorConfigBuilder,
    GovernorConfig,
    GovernorLayer,
};

let governor_conf: GovernorConfig<_, _> = GovernorConfigBuilder::default()
    .per_second(10)
    .burst_size(20)
    .key_extractor(SmartIpKeyExtractor)
    .use_headers()  // 🔑 启用响应头
    .finish()
    .unwrap();
```

响应头包含：
```
X-RateLimit-Limit: 20          # 总配额
X-RateLimit-Remaining: 15      # 剩余配额
X-RateLimit-Reset: 1708000000  # 重置时间戳
Retry-After: 1                 # 限流时，多久后重试
```

---

## 底层 Governor 直接使用

不用 Tower 中间件，手动控制限流：

```rust
use governor::{
    Quota, RateLimiter,
    clock::DefaultClock,
    state::{InMemoryState, NotKeyed},
};
use std::{num::NonZeroU32, sync::Arc};

// 创建限流器：每秒 10 个请求
fn create_limiter() -> Arc<RateLimiter<NotKeyed, InMemoryState, DefaultClock>> {
    let quota = Quota::per_second(NonZeroU32::new(10).unwrap());
    Arc::new(RateLimiter::direct(quota))
}

async fn handle_request(
    limiter: Arc<RateLimiter<NotKeyed, InMemoryState, DefaultClock>>,
) -> Result<String, &'static str> {
    // 尝试获取令牌
    match limiter.check() {
        Ok(_) => Ok("Request processed".to_string()),
        Err(_) => Err("Rate limited"),
    }
}

// 带 key 的限流器（按用户）
use governor::state::keyed::DashMapStateStore;

type KeyedLimiter = RateLimiter<String, DashMapStateStore<String>, DefaultClock>;

fn create_keyed_limiter() -> Arc<KeyedLimiter> {
    let quota = Quota::per_second(NonZeroU32::new(5).unwrap());
    Arc::new(RateLimiter::dashmap(quota))
}

async fn handle_user_request(
    limiter: Arc<KeyedLimiter>,
    user_id: String,
) -> Result<String, &'static str> {
    match limiter.check_key(&user_id) {
        Ok(_) => Ok(format!("OK for user {}", user_id)),
        Err(_) => Err("Rate limited"),
    }
}
```

---

## 分布式限流（Redis）

单机限流不够？用 Redis 实现分布式限流：

```rust
use redis::AsyncCommands;

async fn check_rate_limit(
    redis: &mut redis::aio::MultiplexedConnection,
    key: &str,
    limit: u64,
    window_seconds: u64,
) -> Result<bool, redis::RedisError> {
    let redis_key = format!("ratelimit:{}", key);
    
    // INCR + EXPIRE 原子操作
    let count: u64 = redis.incr(&redis_key, 1).await?;
    
    if count == 1 {
        // 首次请求，设置过期时间
        redis.expire(&redis_key, window_seconds as i64).await?;
    }
    
    Ok(count <= limit)
}

// 使用
let allowed = check_rate_limit(
    &mut redis_conn,
    "user:123",
    100,  // 100 次
    60,   // 每分钟
).await?;

if !allowed {
    return Err(StatusCode::TOO_MANY_REQUESTS);
}
```

---

## 与 Laravel 对比

| 特性 | Laravel | Rust (tower_governor) |
|------|---------|----------------------|
| 配置方式 | `throttle:60,1` | `per_second(60)` |
| Key 提取 | 自动（用户/IP） | `KeyExtractor` trait |
| 存储 | Cache (Redis) | 内存 / Redis |
| 响应码 | 429 | 429 |
| 响应头 | 自动 | `.use_headers()` |

---

## 最佳实践

1. **分层限流**
   - 全局限流防 DDoS
   - 路由级限流防滥用
   - 用户级限流保公平

2. **合理配置**
   - 登录/注册：严格（1-3/秒）
   - 普通 API：适中（50-100/秒）
   - 内部调用：宽松或无限制

3. **返回有用信息**
   - 启用 RateLimit 响应头
   - 告诉用户何时可以重试

4. **监控与调整**
   - 记录被限流的请求
   - 根据实际情况调整阈值

---

## 本课总结

- `governor` 是 Rust 主流限流库，基于令牌桶算法
- `tower_governor` 把它封装成 Tower 中间件，和 Axum 无缝集成
- `SmartIpKeyExtractor` 处理 IP 限流，支持代理场景
- 自定义 `KeyExtractor` 实现按用户/API Key 限流
- 分布式场景用 Redis 做共享存储

---

*性奴001 · Rust 学习小组 · 2026-02-19*
