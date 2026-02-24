# 第 98 课：std::sync — 同步原语

> 日期：2026-02-25

上节课学了 `std::thread` 创建线程，但线程之间如何**安全共享数据**？这就是 `std::sync` 的领域！

---

## 📦 std::sync 核心成员

```rust
use std::sync::{
    Arc,           // 原子引用计数（线程安全的 Rc）
    Mutex,         // 互斥锁
    RwLock,        // 读写锁
    Condvar,       // 条件变量
    Barrier,       // 屏障
    Once,          // 一次性初始化
    atomic::*,     // 原子类型
};
```

---

## 🔒 Mutex — 互斥锁

最常用的同步原语，保证同一时刻只有一个线程访问数据：

```rust
use std::sync::Mutex;

fn main() {
    let counter = Mutex::new(0);
    
    // 获取锁
    {
        let mut num = counter.lock().unwrap();
        *num += 1;
    } // MutexGuard 离开作用域，自动释放锁
    
    println!("Counter: {}", *counter.lock().unwrap()); // 1
}
```

**类比 PHP/Laravel：**
```php
// PHP 用 flock 或 Redis 锁
Cache::lock('counter')->block(5, function () {
    $counter = Cache::get('counter', 0);
    Cache::put('counter', $counter + 1);
});
```

### Mutex 的关键特性

1. **lock()** 返回 `MutexGuard`，离开作用域自动解锁
2. **RAII** 模式 — 忘记手动解锁也没关系
3. **中毒 (Poisoning)** — 线程 panic 时锁会被"毒化"

```rust
use std::sync::Mutex;

fn main() {
    let lock = Mutex::new(42);
    
    // lock() 返回 Result
    match lock.lock() {
        Ok(guard) => println!("获取锁成功: {}", *guard),
        Err(poisoned) => {
            // 另一个线程持有锁时 panic 了
            let guard = poisoned.into_inner();
            println!("锁被毒化，但数据仍可访问: {}", *guard);
        }
    }
    
    // try_lock() 不阻塞
    if let Ok(guard) = lock.try_lock() {
        println!("立即获取到锁: {}", *guard);
    } else {
        println!("锁被占用");
    }
}
```

---

## 🔗 Arc — 原子引用计数

`Rc` 不是线程安全的，跨线程共享要用 `Arc`（Atomic Reference Counted）：

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);
    
    let handles: Vec<_> = (0..3).map(|i| {
        let data = Arc::clone(&data); // 增加引用计数
        thread::spawn(move || {
            println!("线程 {} 看到: {:?}", i, data);
        })
    }).collect();
    
    for h in handles {
        h.join().unwrap();
    }
}
```

**Arc vs Rc：**
| 特性 | Rc | Arc |
|------|-----|------|
| 引用计数 | 非原子 | 原子操作 |
| 线程安全 | ❌ | ✅ |
| 性能 | 更快 | 略慢（原子操作开销）|
| 实现 trait | !Send | Send + Sync |

---

## 🔒🔗 Arc + Mutex — 黄金组合

多线程共享可变数据的标准模式：

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // Arc 让多线程共享，Mutex 保护内部数据
    let counter = Arc::new(Mutex::new(0));
    
    let handles: Vec<_> = (0..10).map(|_| {
        let counter = Arc::clone(&counter);
        thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        })
    }).collect();
    
    for h in handles {
        h.join().unwrap();
    }
    
    println!("最终计数: {}", *counter.lock().unwrap()); // 10
}
```

**记住口诀：Arc 管共享，Mutex 管修改！**

---

## 📖 RwLock — 读写锁

允许多个读者或一个写者：

```rust
use std::sync::RwLock;

fn main() {
    let lock = RwLock::new(5);
    
    // 多个线程可以同时读
    {
        let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap();
        println!("读取: {} {}", *r1, *r2);
    } // 读锁释放
    
    // 写时独占
    {
        let mut w = lock.write().unwrap();
        *w += 1;
        println!("写入后: {}", *w);
    }
}
```

**RwLock vs Mutex：**
- 读多写少 → `RwLock` 更高效
- 写频繁 → `Mutex` 更简单
- 不确定 → 先用 `Mutex`

---

## 🔔 Condvar — 条件变量

线程间的通知机制：

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::thread;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = Arc::clone(&pair);
    
    // 消费者：等待数据准备好
    let consumer = thread::spawn(move || {
        let (lock, cvar) = &*pair2;
        let mut ready = lock.lock().unwrap();
        
        while !*ready {
            println!("消费者：等待数据...");
            ready = cvar.wait(ready).unwrap(); // 释放锁并等待
        }
        
        println!("消费者：数据准备好了！");
    });
    
    // 生产者：准备数据
    thread::sleep(std::time::Duration::from_secs(1));
    let (lock, cvar) = &*pair;
    {
        let mut ready = lock.lock().unwrap();
        *ready = true;
        println!("生产者：数据已准备");
    }
    cvar.notify_one(); // 唤醒等待的线程
    
    consumer.join().unwrap();
}
```

---

## 🚧 Barrier — 屏障同步

让多个线程在某点同步：

```rust
use std::sync::{Arc, Barrier};
use std::thread;

fn main() {
    let barrier = Arc::new(Barrier::new(3)); // 等 3 个线程
    
    let handles: Vec<_> = (0..3).map(|i| {
        let barrier = Arc::clone(&barrier);
        thread::spawn(move || {
            println!("线程 {} 准备就绪", i);
            barrier.wait(); // 等所有线程到达
            println!("线程 {} 开始执行！", i);
        })
    }).collect();
    
    for h in handles {
        h.join().unwrap();
    }
}
```

输出：
```
线程 0 准备就绪
线程 1 准备就绪
线程 2 准备就绪
线程 0 开始执行！
线程 1 开始执行！
线程 2 开始执行！
```

---

## 🎯 Once — 一次性初始化

确保代码只执行一次（懒加载）：

```rust
use std::sync::Once;

static INIT: Once = Once::new();
static mut CONFIG: Option<String> = None;

fn get_config() -> &'static str {
    unsafe {
        INIT.call_once(|| {
            println!("初始化配置（只会打印一次）");
            CONFIG = Some("production".to_string());
        });
        CONFIG.as_ref().unwrap()
    }
}

fn main() {
    println!("{}", get_config()); // 初始化配置...production
    println!("{}", get_config()); // production（不再初始化）
    println!("{}", get_config()); // production
}
```

**更好的方式 — OnceLock（Rust 1.70+）：**
```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<String> = OnceLock::new();

fn get_config() -> &'static str {
    CONFIG.get_or_init(|| {
        println!("初始化！");
        "production".to_string()
    })
}
```

---

## ⚛️ 原子类型预览

`std::sync::atomic` 提供无锁原子操作，下节课详细讲：

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

fn main() {
    let counter = AtomicUsize::new(0);
    
    counter.fetch_add(1, Ordering::SeqCst);
    counter.fetch_add(1, Ordering::SeqCst);
    
    println!("{}", counter.load(Ordering::SeqCst)); // 2
}
```

---

## 📝 本课要点

| 类型 | 用途 | 场景 |
|------|------|------|
| `Arc` | 线程安全引用计数 | 跨线程共享数据 |
| `Mutex` | 互斥锁 | 保护可变数据 |
| `RwLock` | 读写锁 | 读多写少场景 |
| `Condvar` | 条件变量 | 线程间通知 |
| `Barrier` | 屏障 | 多线程同步点 |
| `Once` | 一次性执行 | 懒初始化 |

**黄金组合：`Arc<Mutex<T>>`** — 多线程安全共享可变数据

**常见错误：**
1. 忘记 `Arc::clone` → 用了 move，Arc 被吃掉
2. 死锁 → 同一线程多次 lock 同一个 Mutex
3. 锁粒度太大 → 性能问题

---

*下节课：std::sync::atomic — 原子操作与内存顺序*
