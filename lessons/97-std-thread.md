# 第 97 课：std::thread — 多线程编程基础

> 日期：2026-02-24

Rust 以内存安全著称，而这种安全性在多线程编程中更是大放异彩 — 编译器帮你杜绝数据竞争！

---

## 📦 模块概览

```rust
use std::thread;

// 主要 API
thread::spawn()          // 创建新线程
thread::current()        // 获取当前线程
thread::sleep()          // 休眠
thread::yield_now()      // 让出 CPU
thread::park()           // 阻塞当前线程
thread::Thread::unpark() // 唤醒线程
```

---

## 🚀 创建线程

最简单的方式 — `thread::spawn`：

```rust
use std::thread;
use std::time::Duration;

fn main() {
    // spawn 接受一个闭包
    let handle = thread::spawn(|| {
        for i in 1..=5 {
            println!("子线程: {}", i);
            thread::sleep(Duration::from_millis(100));
        }
    });

    // 主线程继续执行
    for i in 1..=3 {
        println!("主线程: {}", i);
        thread::sleep(Duration::from_millis(150));
    }

    // 等待子线程结束
    handle.join().unwrap();
    println!("所有线程完成！");
}
```

输出（顺序可能不同）：
```
主线程: 1
子线程: 1
子线程: 2
主线程: 2
子线程: 3
主线程: 3
子线程: 4
子线程: 5
所有线程完成！
```

**类比 PHP：**
```php
// PHP 没有原生线程（有 pthreads 扩展）
// 通常用 pcntl_fork 或队列 worker
$pid = pcntl_fork();
if ($pid == 0) {
    // 子进程
}
```

---

## 🔑 JoinHandle — 线程句柄

`spawn` 返回 `JoinHandle<T>`，其中 T 是闭包的返回值：

```rust
use std::thread;

fn main() {
    // 线程可以返回值
    let handle = thread::spawn(|| {
        let sum: u64 = (1..=100).sum();
        sum // 返回计算结果
    });

    // join() 返回 Result<T, ...>
    let result = handle.join().unwrap();
    println!("1+2+...+100 = {}", result); // 5050
}
```

### 多线程并行计算

```rust
use std::thread;

fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8];
    let chunk_size = 2;
    
    let mut handles = vec![];
    
    for chunk in numbers.chunks(chunk_size) {
        let chunk = chunk.to_vec(); // clone 数据给线程
        let handle = thread::spawn(move || {
            let sum: i32 = chunk.iter().sum();
            println!("线程计算: {:?} = {}", chunk, sum);
            sum
        });
        handles.push(handle);
    }
    
    // 收集所有结果
    let total: i32 = handles
        .into_iter()
        .map(|h| h.join().unwrap())
        .sum();
    
    println!("总和: {}", total); // 36
}
```

---

## 📦 move 闭包 — 转移所有权

线程闭包默认借用外部变量，但借用在多线程中很危险（可能悬垂引用）。

**❌ 这样会报错：**
```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    
    thread::spawn(|| {
        println!("{:?}", data); // 借用 data
    }); // 编译错误！
}
```

编译器报错：闭包可能比 `data` 活得更久。

**✅ 用 `move` 转移所有权：**
```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    
    thread::spawn(move || {
        // data 的所有权移入闭包
        println!("{:?}", data);
    }).join().unwrap();
    
    // println!("{:?}", data); // 错误！data 已移走
}
```

---

## 🏷️ 线程命名与标识

```rust
use std::thread;

fn main() {
    // 当前线程信息
    let main_thread = thread::current();
    println!("主线程名: {:?}", main_thread.name()); // Some("main")
    println!("主线程ID: {:?}", main_thread.id());

    // 用 Builder 创建命名线程
    let handle = thread::Builder::new()
        .name("worker-1".to_string())
        .stack_size(4 * 1024 * 1024) // 4MB 栈
        .spawn(|| {
            let me = thread::current();
            println!("我是: {:?}", me.name()); // Some("worker-1")
        })
        .unwrap();

    handle.join().unwrap();
}
```

命名线程的好处：调试时更容易识别！

---

## ⏸️ park / unpark — 线程挂起与唤醒

比 sleep 更精确的控制：

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let worker = thread::spawn(|| {
        println!("Worker: 等待任务...");
        thread::park(); // 挂起，等待唤醒
        println!("Worker: 收到任务，开始工作！");
    });

    thread::sleep(Duration::from_secs(1));
    println!("Main: 分配任务给 worker");
    worker.thread().unpark(); // 唤醒

    worker.join().unwrap();
}
```

**park/unpark vs sleep：**
- `sleep` 固定等待时间
- `park` 等待被 `unpark` 唤醒，更灵活

---

## ⚠️ 线程安全的编译期保证

Rust 用两个 marker trait 保证线程安全：

```rust
// Send: 可以安全地发送到其他线程
// Sync: 可以安全地被多个线程同时引用

// 大多数类型都是 Send + Sync
// 但有些不是：

use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let rc = Rc::new(42);
    
    // thread::spawn(move || {
    //     println!("{}", rc); // 编译错误！Rc 不是 Send
    // });
    
    // Rc 不是线程安全的，编译器阻止你犯错！
    // 多线程要用 Arc（Atomic Reference Counted）
}
```

这就是 Rust 的魔法：**数据竞争在编译期就被消灭了！**

---

## 🧵 thread_local! — 线程局部存储

每个线程有自己独立的变量副本：

```rust
use std::cell::RefCell;
use std::thread;

thread_local! {
    static COUNTER: RefCell<u32> = RefCell::new(0);
}

fn main() {
    COUNTER.with(|c| {
        *c.borrow_mut() += 1;
        println!("主线程: {}", c.borrow()); // 1
    });

    let handle = thread::spawn(|| {
        COUNTER.with(|c| {
            *c.borrow_mut() += 10;
            println!("子线程: {}", c.borrow()); // 10（独立副本）
        });
    });
    handle.join().unwrap();

    COUNTER.with(|c| {
        println!("主线程最终: {}", c.borrow()); // 仍然是 1
    });
}
```

---

## 🔢 可用 CPU 核心数

```rust
use std::thread;

fn main() {
    let cpus = thread::available_parallelism()
        .map(|n| n.get())
        .unwrap_or(1);
    
    println!("可用并行度: {}", cpus);
    
    // 根据 CPU 数量创建线程池
    let handles: Vec<_> = (0..cpus)
        .map(|i| {
            thread::spawn(move || {
                println!("Worker {} 启动", i);
            })
        })
        .collect();
    
    for h in handles {
        h.join().unwrap();
    }
}
```

---

## 📝 本课要点

| API | 用途 |
|-----|------|
| `thread::spawn` | 创建新线程 |
| `handle.join()` | 等待线程结束，获取返回值 |
| `move` 闭包 | 转移所有权给线程 |
| `thread::current()` | 获取当前线程 |
| `thread::sleep()` | 休眠指定时间 |
| `park() / unpark()` | 挂起/唤醒线程 |
| `thread_local!` | 线程局部存储 |
| `available_parallelism()` | 获取 CPU 核心数 |

**Rust 多线程三大法宝：**
1. `move` 闭包 — 转移所有权，避免悬垂引用
2. `Send` / `Sync` — 编译期类型检查
3. 所有权系统 — 同一时刻只有一个可变引用

**常见问题：**
- 忘记 `move` → 编译错误（闭包可能 outlive 数据）
- 用 `Rc` 跨线程 → 编译错误（不是 Send）
- 忘记 `join()` → 主线程退出，子线程被杀

---

*下节课：std::sync — 同步原语（Mutex, Arc, Condvar）*
