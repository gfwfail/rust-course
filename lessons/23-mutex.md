# 第23课：Mutex 互斥锁

Mutex = **Mut**ual **Ex**clusion（互斥锁）

多个线程想访问同一块数据？排队！一次只能一个线程拿到锁。

## 基本用法

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);  // 用 Mutex 包装数据
    
    {
        let mut num = m.lock().unwrap();  // 获取锁
        *num = 6;  // 修改数据
    }  // 锁自动释放（作用域结束）
    
    println!("{:?}", m);  // Mutex { data: 6 }
}
```

## lock() 的注意事项

- `lock()` 返回 `Result<MutexGuard<T>, PoisonError>`
- 如果持有锁的线程 panic，锁就"中毒"了
- 通常用 `.unwrap()` 或 `.expect()` 简化

## 多线程共享：Arc + Mutex

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));  // Arc 包 Mutex
    let mut handles = vec![];
    
    for _ in 0..10 {
        let counter = Arc::clone(&counter);  // 克隆 Arc
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("结果: {}", *counter.lock().unwrap());  // 结果: 10
}
```

## Arc + Mutex 的意义

| 组件 | 作用 |
|------|------|
| Arc | 让多个线程能共享同一数据（引用计数） |
| Mutex | 让共享数据能安全修改（互斥访问） |

两者结合 = **多线程共享可变数据**

## 死锁警告

```rust
// ❌ 经典死锁
// 线程 A: 锁住 m1，等 m2
// 线程 B: 锁住 m2，等 m1
// 两边都在等对方，永远卡住
```

**避免方法**：统一获取锁的顺序！

Rust 不能在编译时阻止死锁，需要自己小心设计。

---

**下节课**：RwLock 读写锁
