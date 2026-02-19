# 第 57 课：优雅关闭 (Graceful Shutdown)

> 生产环境必备技能，确保服务平滑重启

---

## 📖 为什么需要优雅关闭？

想象一下这个场景：你的服务正在处理一个支付请求，用户已经扣款了，这时候运维执行了 `kill` 命令重启服务...

💀 **结果**：用户钱扣了，但订单没创建成功。

这就是为什么我们需要 **Graceful Shutdown**（优雅关闭）：

1. **完成正在处理的请求** — 不要半途而废
2. **停止接受新请求** — 别接新活了
3. **释放资源** — 关闭数据库连接、刷新缓存等
4. **设置超时** — 别无限等待，该杀还是得杀

---

## 🔧 Tokio 中的信号处理

Rust 标准库不直接支持信号处理，但 Tokio 提供了跨平台的信号 API：

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    // 监听 Ctrl+C (SIGINT)
    signal::ctrl_c().await.expect("Failed to listen for Ctrl+C");
    println!("收到关闭信号，开始清理...");
}
```

### Unix 信号（Linux/macOS）

```rust
use tokio::signal::unix::{signal, SignalKind};

async fn shutdown_signal() {
    let mut sigterm = signal(SignalKind::terminate())
        .expect("Failed to create SIGTERM handler");
    let mut sigint = signal(SignalKind::interrupt())
        .expect("Failed to create SIGINT handler");
    
    tokio::select! {
        _ = sigterm.recv() => println!("收到 SIGTERM"),
        _ = sigint.recv() => println!("收到 SIGINT (Ctrl+C)"),
    }
}
```

**常见信号**：
| 信号 | 编号 | 说明 |
|------|------|------|
| `SIGINT` | 2 | Ctrl+C，用户中断 |
| `SIGTERM` | 15 | 优雅终止，`kill` 默认发送 |
| `SIGKILL` | 9 | 强制杀死，无法捕获 |
| `SIGHUP` | 1 | 终端关闭或重载配置 |

---

## 🏗️ Axum 优雅关闭模式

Axum 内置了优雅关闭支持，配合 `tokio::net::TcpListener`：

```rust
use axum::{routing::get, Router};
use tokio::net::TcpListener;
use tokio::signal;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(|| async { "Hello, World!" }))
        .route("/slow", get(slow_handler));

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("🚀 Server running on http://0.0.0.0:3000");

    // 使用 with_graceful_shutdown
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
    
    println!("✅ Server shut down gracefully");
}

async fn shutdown_signal() {
    signal::ctrl_c()
        .await
        .expect("Failed to install Ctrl+C handler");
    println!("🛑 Shutdown signal received, finishing ongoing requests...");
}

async fn slow_handler() -> &'static str {
    // 模拟慢请求
    tokio::time::sleep(Duration::from_secs(5)).await;
    "Done!"
}
```

**测试方法**：
1. 启动服务
2. 用 curl 发起慢请求：`curl http://localhost:3000/slow`
3. 在请求完成前按 Ctrl+C
4. 观察：服务会等待请求完成后才退出

---

## ⏱️ 带超时的优雅关闭

生产环境不能无限等待，需要设置超时：

```rust
use std::time::Duration;
use tokio::time::timeout;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello" }));
    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    // 创建一个 shutdown channel
    let (shutdown_tx, shutdown_rx) = tokio::sync::oneshot::channel::<()>();
    
    // 启动服务
    let server = axum::serve(listener, app)
        .with_graceful_shutdown(async move {
            shutdown_rx.await.ok();
        });
    
    let server_handle = tokio::spawn(server.into_future());

    // 等待关闭信号
    signal::ctrl_c().await.unwrap();
    println!("🛑 Initiating graceful shutdown...");
    
    // 发送关闭信号
    let _ = shutdown_tx.send(());
    
    // 等待服务关闭，最多 30 秒
    match timeout(Duration::from_secs(30), server_handle).await {
        Ok(_) => println!("✅ Server shut down gracefully"),
        Err(_) => {
            println!("⚠️ Shutdown timed out, forcing exit");
            std::process::exit(1);
        }
    }
}
```

---

## 🔄 完整的生产级模式

结合数据库连接池、后台任务等资源的完整示例：

```rust
use axum::{extract::State, routing::get, Router};
use sqlx::PgPool;
use std::{sync::Arc, time::Duration};
use tokio::{signal, sync::watch};

// 应用状态
struct AppState {
    db: PgPool,
    shutdown: watch::Receiver<bool>,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 初始化追踪
    tracing_subscriber::fmt::init();
    
    // 连接数据库
    let db = PgPool::connect("postgres://localhost/mydb").await?;
    tracing::info!("📦 Database connected");
    
    // 创建 shutdown channel (watch 允许多个接收者)
    let (shutdown_tx, shutdown_rx) = watch::channel(false);
    
    let state = Arc::new(AppState {
        db: db.clone(),
        shutdown: shutdown_rx.clone(),
    });
    
    // 启动后台任务
    let bg_shutdown = shutdown_rx.clone();
    let bg_handle = tokio::spawn(background_worker(bg_shutdown));
    
    // 构建路由
    let app = Router::new()
        .route("/health", get(health_check))
        .with_state(state);
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    tracing::info!("🚀 Server listening on :3000");
    
    // 启动服务
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal(shutdown_tx))
        .await?;
    
    // 等待后台任务结束
    tracing::info!("⏳ Waiting for background tasks...");
    let _ = tokio::time::timeout(
        Duration::from_secs(10),
        bg_handle
    ).await;
    
    // 关闭数据库连接池
    tracing::info!("🔌 Closing database connections...");
    db.close().await;
    
    tracing::info!("✅ Shutdown complete");
    Ok(())
}

async fn shutdown_signal(shutdown_tx: watch::Sender<bool>) {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("Ctrl+C handler failed");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("SIGTERM handler failed")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => tracing::info!("Received SIGINT"),
        _ = terminate => tracing::info!("Received SIGTERM"),
    }

    // 广播关闭信号给所有后台任务
    let _ = shutdown_tx.send(true);
}

// 后台任务示例
async fn background_worker(mut shutdown: watch::Receiver<bool>) {
    loop {
        tokio::select! {
            _ = shutdown.changed() => {
                if *shutdown.borrow() {
                    tracing::info!("🛑 Background worker shutting down");
                    break;
                }
            }
            _ = tokio::time::sleep(Duration::from_secs(60)) => {
                tracing::info!("🔄 Background task running...");
                // 执行定期任务
            }
        }
    }
}

async fn health_check() -> &'static str {
    "OK"
}
```

---

## 📋 关键设计模式总结

| 模式 | 用途 | 实现 |
|------|------|------|
| `oneshot` channel | 单次关闭信号 | `tokio::sync::oneshot` |
| `watch` channel | 广播给多个任务 | `tokio::sync::watch` |
| `CancellationToken` | 取消树 | `tokio_util::sync::CancellationToken` |
| `select!` | 监听多个信号 | `tokio::select!` |

### CancellationToken（更优雅的方式）

```rust
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();
let child_token = token.child_token();

// 后台任务
tokio::spawn(async move {
    tokio::select! {
        _ = child_token.cancelled() => {
            println!("Task cancelled");
        }
        _ = do_work() => {
            println!("Work done");
        }
    }
});

// 触发取消（会级联取消所有 child_token）
token.cancel();
```

**Cargo.toml**:
```toml
[dependencies]
tokio-util = { version = "0.7", features = ["rt"] }
```

---

## 🎯 生产环境检查清单

✅ 监听 SIGTERM（K8s/Docker 默认发送）
✅ 设置关闭超时（通常 30s）
✅ 停止接受新连接
✅ 等待正在处理的请求完成
✅ 停止后台任务
✅ 刷新缓存/日志
✅ 关闭数据库连接池
✅ 返回正确的退出码

---

## 🧪 本地测试

```bash
# 1. 启动服务
cargo run

# 2. 发起慢请求
curl http://localhost:3000/slow &

# 3. 发送 SIGTERM
kill -TERM $(pgrep -f "your_binary")

# 4. 观察日志，确认请求完成后才退出
```

---

## 💡 常见陷阱

**❌ 忘记处理 SIGTERM**
```rust
// K8s 发送 SIGTERM，你只监听了 Ctrl+C
signal::ctrl_c().await; // ❌
```

**✅ 同时监听 SIGINT 和 SIGTERM**
```rust
tokio::select! {
    _ = signal::ctrl_c() => {},
    _ = sigterm.recv() => {},
}
```

**❌ 无超时等待**
```rust
server_handle.await; // 可能永远等下去
```

**✅ 带超时**
```rust
timeout(Duration::from_secs(30), server_handle).await;
```

---

## 📚 延伸阅读

- [Tokio Tutorial: Graceful Shutdown](https://tokio.rs/tokio/topics/shutdown)
- [Axum Examples: Graceful Shutdown](https://github.com/tokio-rs/axum/blob/main/examples/graceful-shutdown/src/main.rs)

---

*课程日期：2026-02-19*
