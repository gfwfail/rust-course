# 第27课：async/await 入门

今天进入异步世界！

## 为什么需要异步？

想象你开了一家餐厅：

**同步模式**（服务员等上菜）：
```rust
fn serve_table() {
    take_order();       // 1分钟
    wait_for_food();    // 10分钟（干等）
    deliver_food();     // 1分钟
}
// 一个服务员 = 12分钟一桌
```

**异步模式**（等菜时服务别桌）：
```rust
async fn serve_table() {
    take_order();             
    wait_for_food().await;    // 等待时去服务别桌！
    deliver_food();           
}
```

## 核心概念：Future

`async` 函数返回一个 `Future`：

```rust
async fn hello() -> String {
    "Hello, async!".to_string()
}

// 等价于
fn hello() -> impl Future<Output = String> { ... }
```

**关键**：Future 是懒惰的，定义时不执行，`.await` 时才执行。

## 基本语法

```rust
async fn fetch_data() -> String {
    "data from server".to_string()
}

async fn main_logic() {
    let data = fetch_data().await;  // 等待结果
    println!("Got: {}", data);
}
```

## 大坑：不 await 就不执行！

```rust
async fn say_hello() {
    println!("Hello!");
}

async fn main_logic() {
    say_hello();        // ❌ 什么都不打印！
    say_hello().await;  // ✅ 打印 "Hello!"
}
```

## 运行时：Tokio

异步代码需要「执行器」来驱动：

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
#[tokio::main]
async fn main() {
    let result = fetch_data().await;
    println!("{}", result);
}
```

## 并发执行：join!

```rust
use tokio::join;
use std::time::Duration;

async fn task_a() -> i32 { 
    tokio::time::sleep(Duration::from_secs(2)).await;
    1 
}

async fn task_b() -> i32 { 
    tokio::time::sleep(Duration::from_secs(2)).await;
    2 
}

#[tokio::main]
async fn main() {
    // 并发执行：只需2秒！
    let (a, b) = join!(task_a(), task_b());
    println!("a={}, b={}", a, b);
}
```

## async vs 线程

| 特性 | 线程 | async |
|------|------|-------|
| 调度 | OS | 运行时 |
| 开销 | 大（~8KB） | 小（~几百字节） |
| 切换 | 任意时刻 | .await 点 |
| 数量 | 几百个 | 几十万个 |
| 适合 | CPU 密集 | I/O 密集 |

## 要点总结

1. `async fn` 返回 Future，调用时不执行
2. `.await` 才会真正执行并等待
3. 需要 runtime（tokio）来驱动
4. `join!` 并发执行多个 Future
5. 异步适合 I/O 密集型（网络、文件）

---

## 后续学习

- async/await 进阶：spawn、select!、超时
- Web 框架：Axum、Actix-web
- 数据库：SQLx、SeaORM
- 实战项目
