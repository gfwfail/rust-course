# 第 53 课：Tower 中间件 (tower)

> 授课时间：2026-02-19  
> 主题：Tower 中间件机制与 Axum 集成

---

## 📚 为什么要学 Tower？

如果你用过 Axum，你其实已经在用 Tower 了！Axum 整个框架都是基于 Tower 构建的。

Tower 是 Rust 生态中处理请求/响应的抽象层，类似于：
- Laravel 的 Middleware
- Express 的 middleware
- Koa 的洋葱模型

理解 Tower = 掌握 Axum 中间件的精髓。

---

## 🎯 核心概念：Service trait

```rust
// Tower 的核心就是这个 trait
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;
    
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

**简单理解：**
- `Request` 进来
- `Response` 出去
- 中间可以做任何事

---

## 🛠️ 实战：给 Axum 添加自定义中间件

### 1️⃣ 使用内置中间件

```rust
use axum::{Router, routing::get};
use tower_http::{
    cors::CorsLayer,
    trace::TraceLayer,
    timeout::TimeoutLayer,
    compression::CompressionLayer,
};
use std::time::Duration;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(handler))
        // 添加中间件（从下往上执行）
        .layer(CompressionLayer::new())      // 4. 压缩响应
        .layer(TimeoutLayer::new(Duration::from_secs(30)))  // 3. 超时
        .layer(TraceLayer::new_for_http())   // 2. 日志追踪
        .layer(CorsLayer::permissive());     // 1. CORS
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> &'static str {
    "Hello, Tower!"
}
```

---

### 2️⃣ 用闭包快速创建中间件

```rust
use axum::{
    Router, routing::get,
    middleware::{self, Next},
    extract::Request,
    response::Response,
};

// 最简单的方式：用 async 函数
async fn logging_middleware(
    request: Request,
    next: Next,
) -> Response {
    let method = request.method().clone();
    let uri = request.uri().clone();
    
    // 请求前
    println!("➡️  {} {}", method, uri);
    let start = std::time::Instant::now();
    
    // 调用下一个中间件/处理器
    let response = next.run(request).await;
    
    // 响应后
    println!("⬅️  {} {} - {:?}", method, uri, start.elapsed());
    
    response
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(|| async { "Hello!" }))
        .layer(middleware::from_fn(logging_middleware));
    
    // ...
}
```

---

### 3️⃣ 带状态的中间件

```rust
use axum::{
    Router, routing::get,
    middleware::{self, Next},
    extract::{Request, State},
    response::{Response, IntoResponse},
    http::StatusCode,
};
use std::sync::Arc;
use tokio::sync::RwLock;
use std::collections::HashMap;

// 简易限流器状态
#[derive(Clone, Default)]
struct RateLimiter {
    requests: Arc<RwLock<HashMap<String, u32>>>,
}

async fn rate_limit_middleware(
    State(limiter): State<RateLimiter>,
    request: Request,
    next: Next,
) -> Response {
    // 获取客户端 IP（简化版）
    let ip = request
        .headers()
        .get("x-forwarded-for")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string();
    
    // 检查请求次数
    {
        let mut requests = limiter.requests.write().await;
        let count = requests.entry(ip.clone()).or_insert(0);
        *count += 1;
        
        if *count > 100 {
            return (
                StatusCode::TOO_MANY_REQUESTS,
                "Rate limit exceeded"
            ).into_response();
        }
    }
    
    next.run(request).await
}

#[tokio::main]
async fn main() {
    let limiter = RateLimiter::default();
    
    let app = Router::new()
        .route("/", get(|| async { "Hello!" }))
        .layer(middleware::from_fn_with_state(
            limiter.clone(),
            rate_limit_middleware,
        ))
        .with_state(limiter);
    
    // ...
}
```

---

### 4️⃣ 认证中间件（实战常用）

```rust
use axum::{
    Router, routing::get,
    middleware::{self, Next},
    extract::Request,
    response::{Response, IntoResponse},
    http::{StatusCode, header},
    Extension,
};

#[derive(Clone)]
struct CurrentUser {
    id: i64,
    name: String,
}

async fn auth_middleware(
    mut request: Request,
    next: Next,
) -> Response {
    // 从 Header 获取 token
    let auth_header = request
        .headers()
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok());
    
    let token = match auth_header {
        Some(h) if h.starts_with("Bearer ") => &h[7..],
        _ => {
            return (
                StatusCode::UNAUTHORIZED,
                "Missing or invalid Authorization header"
            ).into_response();
        }
    };
    
    // 验证 token（这里简化，实际用 JWT）
    let user = match validate_token(token).await {
        Ok(user) => user,
        Err(_) => {
            return (
                StatusCode::UNAUTHORIZED,
                "Invalid token"
            ).into_response();
        }
    };
    
    // 将用户信息注入请求
    request.extensions_mut().insert(user);
    
    next.run(request).await
}

async fn validate_token(token: &str) -> Result<CurrentUser, ()> {
    // 实际项目这里解析 JWT
    if token == "valid_token" {
        Ok(CurrentUser { id: 1, name: "Alice".into() })
    } else {
        Err(())
    }
}

// 在 handler 中获取用户
async fn protected_handler(
    Extension(user): Extension<CurrentUser>,
) -> String {
    format!("Hello, {}!", user.name)
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        // 需要认证的路由
        .route("/protected", get(protected_handler))
        .layer(middleware::from_fn(auth_middleware))
        // 公开路由（不经过中间件）
        .route("/public", get(|| async { "Public!" }));
    
    // ...
}
```

---

## 🎨 中间件执行顺序

```rust
Router::new()
    .route("/", get(handler))
    .layer(A)  // 最后添加，最先执行
    .layer(B)  // 
    .layer(C)  // 最先添加，最后执行
```

**请求流程：**
```
Request → A → B → C → Handler → C → B → A → Response
         ↓                              ↑
       （洋葱模型，像 Laravel/Koa）
```

---

## 📦 tower-http 常用中间件

```toml
[dependencies]
tower-http = { version = "0.6", features = ["full"] }
```

| 中间件 | 用途 |
|--------|------|
| `TraceLayer` | 请求追踪/日志 |
| `CorsLayer` | CORS 跨域 |
| `CompressionLayer` | Gzip/Brotli 压缩 |
| `TimeoutLayer` | 请求超时 |
| `RequestIdLayer` | 请求 ID |
| `CatchPanicLayer` | 捕获 panic |
| `SetRequestHeaderLayer` | 设置请求头 |
| `SetResponseHeaderLayer` | 设置响应头 |
| `NormalizePath` | 路径标准化 |

---

## 💡 最佳实践

```rust
use axum::Router;
use tower_http::{
    cors::{CorsLayer, Any},
    trace::TraceLayer,
    timeout::TimeoutLayer,
    catch_panic::CatchPanicLayer,
    request_id::{MakeRequestUuid, RequestIdLayer, PropagateRequestIdLayer},
    compression::CompressionLayer,
};
use std::time::Duration;

fn create_app() -> Router {
    Router::new()
        .route("/", get(handler))
        // 中间件从下往上执行
        .layer(CompressionLayer::new())
        .layer(PropagateRequestIdLayer::x_request_id())
        .layer(RequestIdLayer::x_request_id(MakeRequestUuid))
        .layer(TraceLayer::new_for_http())
        .layer(TimeoutLayer::new(Duration::from_secs(30)))
        .layer(CatchPanicLayer::new())
        .layer(
            CorsLayer::new()
                .allow_origin(Any)
                .allow_methods(Any)
                .allow_headers(Any)
        )
}
```

---

## 🧠 核心要点

1. **Tower 是 Service trait 的抽象**，请求进响应出
2. **Axum 的 layer() 就是 Tower 中间件**
3. **执行顺序是洋葱模型**，最后 layer 的最先执行
4. **用 `middleware::from_fn` 快速创建**，不用手写 Service
5. **tower-http 提供了大量现成中间件**，别重复造轮子

---

## 📖 延伸阅读

- [Tower 官方文档](https://docs.rs/tower/latest/tower/)
- [tower-http 文档](https://docs.rs/tower-http/latest/tower_http/)
- [Axum middleware 指南](https://docs.rs/axum/latest/axum/middleware/index.html)

---

*笔记整理：性奴001*
