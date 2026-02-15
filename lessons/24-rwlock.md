# 第24课：RwLock 读写锁

Mutex 太粗暴：同时只能一个线程。但很多场景读多写少，用 Mutex 太浪费。

RwLock 允许：
- **多个读者**同时读取（共享锁）
- **只有一个写者**独占写入（排他锁）
- 读和写互斥

## 基本用法

```rust
use std::sync::RwLock;

fn main() {
    let lock = RwLock::new(5);
    
    // 读取：可以有多个同时读
    {
        let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap(); // ✅ 同时读没问题
        println!("r1 = {}, r2 = {}", r1, r2);
    }
    
    // 写入：独占访问
    {
        let mut w = lock.write().unwrap();
        *w += 1;
    }
}
```

## 多线程配合 Arc

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));
    let mut handles = vec![];
    
    // 启动3个读线程
    for i in 0..3 {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let r = data.read().unwrap();
            println!("读者{}: {:?}", i, *r);
        }));
    }
    
    // 启动1个写线程
    {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let mut w = data.write().unwrap();
            w.push(4);
        }));
    }
    
    for h in handles {
        h.join().unwrap();
    }
}
```

## 写者饥饿问题

如果读者源源不断，写者可能一直拿不到锁！

**解决方案**：
1. 控制读的频率，定期释放读锁
2. 用 `parking_lot` crate 的 RwLock（公平调度）

## Mutex vs RwLock

| 场景 | 选择 |
|------|------|
| 读多写少（配置、缓存） | RwLock |
| 读写频率差不多 | Mutex |
| 只需要写 | Mutex |
| 持有时间很短 | Mutex（锁切换有开销） |

**经验**：不确定时先用 Mutex，性能有问题再考虑 RwLock。

---

**下节课**：Send 与 Sync
