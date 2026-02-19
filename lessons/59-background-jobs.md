# 第59课：后台任务与定时调度 (Background Jobs & Cron)

## 为什么需要后台任务？

Web 应用中有些工作不能阻塞请求：
- 发邮件、推送通知
- 数据同步、报表生成
- 定时清理、心跳检测
- 重试失败的操作

PHP 方案：Laravel Queue + Horizon
Rust 方案：Tokio spawn + 定时调度器

---

## 1️⃣ 最简单的后台任务：tokio::spawn

```rust
use tokio::time::{sleep, Duration};

async fn send_email(to: &str, content: &str) {
    // 模拟耗时操作
    sleep(Duration::from_secs(2)).await;
    println!("📧 Email sent to {}: {}", to, content);
}

#[tokio::main]
async fn main() {
    // 不等待，直接在后台跑
    tokio::spawn(async {
        send_email("user@example.com", "Welcome!").await;
    });

    println!("👋 Main continues immediately");
    
    // 等一下，让后台任务完成
    sleep(Duration::from_secs(3)).await;
}
```

**输出：**
```
👋 Main continues immediately
📧 Email sent to user@example.com: Welcome!
```

---

## 2️⃣ 带错误处理的 spawn

```rust
use tokio::task::JoinHandle;

async fn risky_task() -> Result<String, &'static str> {
    // 模拟可能失败的操作
    Err("Connection refused")
}

#[tokio::main]
async fn main() {
    let handle: JoinHandle<Result<String, &'static str>> = 
        tokio::spawn(async {
            risky_task().await
        });

    // 等待结果
    match handle.await {
        Ok(Ok(result)) => println!("✅ Success: {}", result),
        Ok(Err(e)) => println!("❌ Task error: {}", e),
        Err(e) => println!("💥 Task panicked: {}", e),
    }
}
```

**注意：** `JoinHandle.await` 返回 `Result<T, JoinError>`，套两层！

---

## 3️⃣ 定时调度：tokio-cron-scheduler

这是 Rust 版的 Cron，支持标准 cron 表达式。

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-cron-scheduler = "0.10"
```

```rust
use tokio_cron_scheduler::{Job, JobScheduler};
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建调度器
    let sched = JobScheduler::new().await?;

    // 每分钟执行一次
    sched.add(
        Job::new_async("0 * * * * *", |_uuid, _lock| {
            Box::pin(async {
                println!("⏰ Every minute: {:?}", chrono::Local::now());
            })
        })?
    ).await?;

    // 每5秒执行一次（用 Duration 更直观）
    sched.add(
        Job::new_repeated_async(
            Duration::from_secs(5),
            |_uuid, _lock| {
                Box::pin(async {
                    println!("🔄 Every 5 seconds");
                })
            }
        )?
    ).await?;

    // 启动调度器
    sched.start().await?;

    // 保持运行
    tokio::signal::ctrl_c().await?;
    sched.shutdown().await?;
    
    Ok(())
}
```

---

## 4️⃣ Cron 表达式速查

```
┌──────────── 秒 (0-59)
│ ┌────────── 分 (0-59)
│ │ ┌──────── 时 (0-23)
│ │ │ ┌────── 日 (1-31)
│ │ │ │ ┌──── 月 (1-12)
│ │ │ │ │ ┌── 星期 (0-6, 0=周日)
│ │ │ │ │ │
* * * * * *
```

常用示例：
- `0 0 * * * *` → 每小时整点
- `0 30 9 * * *` → 每天 9:30
- `0 0 0 * * 1` → 每周一凌晨
- `0 */5 * * * *` → 每5分钟

---

## 5️⃣ 在 Axum 中集成后台任务

```rust
use axum::{Router, routing::post, Json, Extension};
use tokio::sync::mpsc;
use std::sync::Arc;

// 任务消息
enum BackgroundTask {
    SendEmail { to: String, content: String },
    SyncData { user_id: i64 },
}

// 任务处理器
async fn task_worker(mut rx: mpsc::Receiver<BackgroundTask>) {
    while let Some(task) = rx.recv().await {
        match task {
            BackgroundTask::SendEmail { to, content } => {
                println!("📧 Sending to {}: {}", to, content);
                // 实际发送逻辑...
            }
            BackgroundTask::SyncData { user_id } => {
                println!("🔄 Syncing user {}", user_id);
                // 实际同步逻辑...
            }
        }
    }
}

// API Handler
async fn register_user(
    Extension(tx): Extension<mpsc::Sender<BackgroundTask>>,
) -> &'static str {
    // 发送后台任务，不等待
    let _ = tx.send(BackgroundTask::SendEmail {
        to: "new@user.com".to_string(),
        content: "Welcome!".to_string(),
    }).await;

    "User registered!" // 立即返回
}

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel::<BackgroundTask>(100);

    // 启动后台 worker
    tokio::spawn(task_worker(rx));

    let app = Router::new()
        .route("/register", post(register_user))
        .layer(Extension(tx));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

## 6️⃣ 重试机制

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn with_retry<F, Fut, T, E>(
    mut f: F,
    max_retries: u32,
    delay: Duration,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
    E: std::fmt::Debug,
{
    let mut attempts = 0;
    
    loop {
        attempts += 1;
        match f().await {
            Ok(result) => return Ok(result),
            Err(e) if attempts < max_retries => {
                println!("⚠️ Attempt {} failed: {:?}, retrying...", attempts, e);
                sleep(delay * attempts).await; // 指数退避
            }
            Err(e) => return Err(e),
        }
    }
}

// 使用
async fn flaky_api_call() -> Result<String, &'static str> {
    // 模拟不稳定的 API
    Err("timeout")
}

#[tokio::main]
async fn main() {
    let result = with_retry(
        || flaky_api_call(),
        3,
        Duration::from_secs(1),
    ).await;
    
    println!("Final result: {:?}", result);
}
```

---

## 📝 总结

| 场景 | 方案 |
|------|------|
| 简单异步任务 | `tokio::spawn` |
| 需要结果 | `JoinHandle.await` |
| 定时任务 | `tokio-cron-scheduler` |
| 任务队列 | `mpsc::channel` + worker |
| 失败重试 | 自定义 retry 函数 |

**最佳实践：**
1. 后台任务要有超时保护
2. 失败任务要有重试 + 告警
3. 长时间任务考虑拆分
4. 生产环境用持久化队列（Redis、RabbitMQ）

---

## 📚 延伸阅读

- [tokio-cron-scheduler](https://docs.rs/tokio-cron-scheduler)
- [Tokio Spawning](https://tokio.rs/tokio/tutorial/spawning)
- [backon - Retry with backoff](https://docs.rs/backon)

---

*课程日期：2026-02-20*
