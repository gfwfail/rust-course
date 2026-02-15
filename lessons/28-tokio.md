# 第28课：Tokio 运行时入门

> 🕐 授课时间：2026-02-16 06:00  
> 📍 地点：Telegram 群 Rust学习小组

---

## 为什么需要运行时？

`async fn` 返回的是一个 `Future`，它只是一个"待执行的计划"：

```rust
async fn hello() -> String {
    "Hello".to_string()
}

fn main() {
    let future = hello(); // ❌ 这里什么都没执行
    // future 只是一个 Future，需要 runtime 去 poll 它
}
```

运行时负责：
1. **调度**：决定哪个 Future 该执行
2. **轮询**：调用 `.poll()` 推进 Future
3. **I/O 事件循环**：监听网络、文件等事件
4. **线程池管理**：分配工作到多核

---

## 最简单的 Tokio 程序

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
#[tokio::main]
async fn main() {
    println!("Hello from Tokio!");
}
```

`#[tokio::main]` 宏展开后是：

```rust
fn main() {
    tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async {
            println!("Hello from Tokio!");
        })
}
```

---

## 异步定时器

Tokio 提供 `tokio::time`，代替标准库的 `std::thread::sleep`：

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("开始...");
    sleep(Duration::from_secs(2)).await;  // 不阻塞线程！
    println!("2秒后...");
}
```

⚠️ **绝对不要在 async 里用 `std::thread::sleep`**！
它会阻塞整个线程，拖慢其他任务。

---

## 并发执行多个任务

```rust
use tokio::time::{sleep, Duration};

async fn task_a() {
    sleep(Duration::from_secs(2)).await;
    println!("Task A 完成");
}

async fn task_b() {
    sleep(Duration::from_secs(1)).await;
    println!("Task B 完成");
}

#[tokio::main]
async fn main() {
    // join! 并发执行，等待全部完成
    tokio::join!(task_a(), task_b());
    println!("全部完成");
}
```

输出顺序：
```
Task B 完成  (1秒后)
Task A 完成  (2秒后)
全部完成
```

注意：是 **并发 (concurrent)** 不是 **并行 (parallel)**。
在单线程上交替执行，不是同时占用多核。

---

## spawn：后台任务

`tokio::spawn` 创建一个独立任务，不等待它完成：

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // 启动后台任务
    let handle = tokio::spawn(async {
        sleep(Duration::from_secs(2)).await;
        println!("后台任务完成");
        42  // 返回值
    });
    
    println!("主任务继续执行...");
    
    // 需要结果时再 await
    let result = handle.await.unwrap();
    println!("后台任务返回: {}", result);
}
```

---

## select!：竞速执行

`tokio::select!` 等待多个 Future，**只执行最先完成的那个**：

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = sleep(Duration::from_secs(1)) => {
            println!("定时器先完成");
        }
        _ = some_network_call() => {
            println!("网络请求先完成");
        }
    }
    // 只有一个分支会执行
}
```

经典用法：**超时控制**

```rust
use tokio::time::{timeout, Duration};

async fn slow_operation() -> String {
    tokio::time::sleep(Duration::from_secs(10)).await;
    "done".to_string()
}

#[tokio::main]
async fn main() {
    match timeout(Duration::from_secs(3), slow_operation()).await {
        Ok(result) => println!("成功: {}", result),
        Err(_) => println!("超时了！"),
    }
}
```

---

## 多线程 vs 单线程运行时

```rust
// 多线程（默认）- 利用多核
#[tokio::main]
async fn main() { }

// 单线程 - 更低开销，适合轻量场景
#[tokio::main(flavor = "current_thread")]
async fn main() { }

// 指定线程数
#[tokio::main(worker_threads = 4)]
async fn main() { }
```

---

## 💡 对比 PHP/Laravel

| PHP | Tokio |
|-----|-------|
| `sleep(2)` 阻塞进程 | `sleep().await` 释放线程 |
| 每个请求一个进程 | 一个进程处理成千上万连接 |
| Swoole 协程 | Tokio 任务 |
| Laravel Octane | Axum + Tokio |

---

## 🧠 小结

1. **async 需要运行时**：Rust 标准库没有，Tokio 是最流行的选择
2. **#[tokio::main]**：最简单的启动方式
3. **join!**：并发等待多个任务全部完成
4. **spawn**：启动后台任务
5. **select!**：竞速执行，取最快的
6. **timeout**：给异步操作加超时

---

*下节课：用 Axum 搭建第一个 Web 服务*
