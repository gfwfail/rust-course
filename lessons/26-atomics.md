# 第26课：原子类型 Atomics

原子操作是不可分割的操作——要么完全执行，要么完全不执行。

## 为什么需要原子操作？

```rust
// 普通操作不是原子的！
counter += 1;
// 实际分解为：读取 -> 加1 -> 写回
// 多线程下可能出问题
```

## 原子类型

```rust
use std::sync::atomic::{AtomicBool, AtomicI32, AtomicUsize, Ordering};
```

常用原子类型：
- `AtomicBool` — 原子布尔
- `AtomicI32`, `AtomicU32` — 原子整数
- `AtomicUsize` — 原子 usize（常用于计数器）
- `AtomicPtr<T>` — 原子指针

## 基本操作

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

fn main() {
    let counter = AtomicUsize::new(0);
    
    // 存储（写入）
    counter.store(42, Ordering::SeqCst);
    
    // 加载（读取）
    let value = counter.load(Ordering::SeqCst);
    println!("value: {}", value); // 42
    
    // 原子加法，返回旧值
    let old = counter.fetch_add(10, Ordering::SeqCst);
    println!("old: {}, new: {}", old, counter.load(Ordering::SeqCst));
    // old: 42, new: 52
}
```

## 常用方法

```rust
load(ordering)           // 读取当前值
store(val, ordering)     // 写入新值
fetch_add(val, ordering) // 原子加
fetch_sub(val, ordering) // 原子减
swap(val, ordering)      // 交换，返回旧值
compare_exchange(...)    // 比较并交换（CAS）
```

## Ordering（内存顺序）

```rust
Ordering::Relaxed   // 最宽松，只保证原子性
Ordering::Acquire   // 读操作
Ordering::Release   // 写操作
Ordering::AcqRel    // 读写都有
Ordering::SeqCst    // 最严格，最安全
```

**新手建议：先用 `SeqCst`，能跑通再优化！**

## 线程安全计数器

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicUsize::new(0));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..1000 {
                counter.fetch_add(1, Ordering::SeqCst);
            }
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Final: {}", counter.load(Ordering::SeqCst));
    // Final: 10000 ✅
}
```

## 原子 vs Mutex

| 特性 | 原子类型 | Mutex |
|------|---------|-------|
| 复杂度 | 简单（单值） | 复杂（任意数据） |
| 性能 | 更快（无锁） | 有锁开销 |
| 阻塞 | 不阻塞 | 可能阻塞 |
| 用途 | 计数器、标志位 | 复杂数据结构 |

---

**下节课**：async/await 入门
