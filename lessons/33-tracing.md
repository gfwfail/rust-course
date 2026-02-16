# 第 33 课：Tracing 日志与追踪

> 📅 2026-02-16 21:00 (Sydney)

## 为什么不用 log？

PHP 的 Laravel 用 Monolog，JS 用 console.log，Rust 有 `log` crate。但在异步/并发场景下，`log` 不够用：

```
[2026-02-16 21:00:00] 处理请求开始
[2026-02-16 21:00:00] 查询数据库
[2026-02-16 21:00:00] 处理请求开始  ← 另一个请求？还是同一个？
[2026-02-16 21:00:00] 查询数据库完成
```

**问题**：多个请求同时进行，日志混在一起，分不清谁是谁。

`tracing` 的解决方案：**Span（跨度）**

---

## 核心概念

### 1. Event（事件）= 传统日志

```rust
use tracing::{info, warn, error, debug, trace};

info!("服务启动");
warn!(port = 8080, "端口被占用");
error!(?err, "请求失败");  // ?err 自动 Debug 格式化
```

### 2. Span（跨度）= 上下文追踪

```rust
use tracing::{info_span, instrument};

// 方式一：手动创建 span
let span = info_span!("process_request", request_id = %id);
let _guard = span.enter();  // 进入 span
// ... 这里的所有日志都会带上 request_id
// _guard drop 时自动退出

// 方式二：用宏（推荐）
#[instrument]
async fn handle_request(id: u64) {
    info!("开始处理");  // 自动带上 id
    query_db().await;
    info!("处理完成");
}
```

---

## 添加依赖

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

---

## 基础配置

```rust
use tracing_subscriber::{fmt, EnvFilter};

fn main() {
    // 最简单的配置
    tracing_subscriber::fmt::init();
    
    // 带环境变量过滤的配置
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .init();
}
```

运行时控制日志级别：
```bash
RUST_LOG=debug cargo run
RUST_LOG=my_app=debug,sqlx=warn cargo run
```

---

## 实战：Web 应用日志

```rust
use axum::{Router, routing::get, extract::Path};
use tracing::{info, instrument, Level};
use tracing_subscriber::{fmt, EnvFilter};

#[tokio::main]
async fn main() {
    // 初始化日志
    tracing_subscriber::fmt()
        .with_max_level(Level::DEBUG)
        .with_target(false)  // 不显示模块路径
        .init();

    info!("🚀 服务启动");

    let app = Router::new()
        .route("/users/:id", get(get_user));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    
    axum::serve(listener, app).await.unwrap();
}

#[instrument(skip_all, fields(user_id = %id))]
async fn get_user(Path(id): Path<u64>) -> String {
    info!("查询用户");
    
    // 模拟数据库查询
    let user = fetch_from_db(id).await;
    
    info!(found = user.is_some(), "查询完成");
    
    user.unwrap_or_else(|| "Not found".to_string())
}

#[instrument]
async fn fetch_from_db(id: u64) -> Option<String> {
    tokio::time::sleep(std::time::Duration::from_millis(50)).await;
    Some(format!("User {}", id))
}
```

输出效果：
```
2026-02-16T10:00:00 INFO 🚀 服务启动
2026-02-16T10:00:01 INFO get_user{user_id=42}: 查询用户
2026-02-16T10:00:01 INFO get_user{user_id=42}:fetch_from_db{id=42}: 进入
2026-02-16T10:00:01 INFO get_user{user_id=42}: 查询完成 found=true
```

注意 `get_user{user_id=42}` —— 所有嵌套调用都带上了上下文！

---

## #[instrument] 详解

```rust
// 基础用法 - 自动记录所有参数
#[instrument]
fn foo(a: i32, b: &str) {}

// 跳过某些参数（比如敏感数据）
#[instrument(skip(password))]
fn login(username: &str, password: &str) {}

// 跳过所有参数，手动指定 fields
#[instrument(skip_all, fields(user_id = %user.id))]
fn process(user: &User, data: &[u8]) {}

// 自定义 span 名称和级别
#[instrument(name = "db_query", level = "debug")]
async fn query() {}

// 记录返回值
#[instrument(ret)]
fn calculate() -> i32 { 42 }

// 记录错误（Result 类型）
#[instrument(err)]
fn might_fail() -> Result<(), Error> { Ok(()) }
```

---

## 结构化日志字段

```rust
// 普通值
info!(port = 8080, "启动");

// Display 格式（%）
info!(user = %username, "登录");

// Debug 格式（?）
info!(config = ?config, "加载配置");

// 空字段（稍后填充）
let span = info_span!("request", response_code = tracing::field::Empty);
// ... 处理后
span.record("response_code", 200);
```

---

## 与 Axum 集成：tower-http

```rust
use axum::{Router, routing::get};
use tower_http::trace::TraceLayer;
use tracing::Level;

let app = Router::new()
    .route("/", get(|| async { "Hello" }))
    .layer(TraceLayer::new_for_http());  // 自动记录请求日志
```

自动生成的日志：
```
DEBUG request{method=GET uri=/ version=HTTP/1.1}: started
DEBUG request{...}: finished status=200 latency=1ms
```

---

## JSON 格式日志（生产环境）

```rust
use tracing_subscriber::fmt::format::FmtSpan;

tracing_subscriber::fmt()
    .json()  // 输出 JSON 格式
    .with_span_events(FmtSpan::CLOSE)  // 记录 span 结束
    .init();
```

输出：
```json
{"timestamp":"2026-02-16T10:00:00","level":"INFO","message":"查询用户","target":"my_app","span":{"user_id":42}}
```

这种格式方便 ELK、Datadog 等日志系统采集。

---

## 常用日志级别

| 级别 | 用途 | 生产环境 |
|------|------|----------|
| ERROR | 需要立即关注的错误 | ✅ |
| WARN | 潜在问题 | ✅ |
| INFO | 重要业务事件 | ✅ |
| DEBUG | 调试信息 | ❌ |
| TRACE | 超详细追踪 | ❌ |

---

## 对比其他语言

| 特性 | Laravel (Monolog) | Node.js (Winston) | Rust (tracing) |
|------|-------------------|-------------------|----------------|
| 基础日志 | ✅ | ✅ | ✅ |
| 结构化字段 | ✅ | ✅ | ✅ |
| **Span 追踪** | ❌ | ❌ | ✅ |
| **零成本抽象** | ❌ | ❌ | ✅ |
| 异步安全 | N/A | ⚠️ | ✅ |

`tracing` 的 Span 是杀手锏：在高并发场景下，能追踪一个请求的完整生命周期。

---

## 今日要点

1. **Event** = 单条日志，**Span** = 上下文跨度
2. `#[instrument]` 自动为函数创建 span
3. 支持结构化字段：`%` Display，`?` Debug
4. `RUST_LOG` 环境变量控制日志级别
5. 生产环境用 JSON 格式输出

---

## 课后练习

给你的 Axum 应用加上 tracing：
1. 初始化 tracing-subscriber
2. 为 handler 添加 `#[instrument]`
3. 用 `RUST_LOG=debug` 运行，观察日志

---

*下节课：Tower 中间件*
