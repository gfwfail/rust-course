# 第25课：Send 与 Sync

Rust 「无畏并发」的幕后英雄：**Send** 和 **Sync** 标记 Trait。

## 为什么需要它们？

Rust 承诺编译时防止数据竞争。编译器怎么知道一个类型能不能跨线程使用？

答案就是这两个 marker trait。

## Send：可以跨线程「移动」

**含义**：类型 `T: Send` 表示 `T` 的值可以安全地从一个线程移动到另一个线程。

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3]; // Vec<i32> 实现了 Send
    
    // ✅ 可以 move 到新线程
    let handle = thread::spawn(move || {
        println!("{:?}", data);
    });
    
    handle.join().unwrap();
}
```

**是 Send 的类型**：
- 基本类型：i32, bool, f64 等
- String, Vec, HashMap
- Arc<T>（如果 T: Send + Sync）
- Mutex<T>（如果 T: Send）

**不是 Send 的类型**：
- `Rc<T>` — 非原子引用计数，跨线程会数据竞争
- `*const T`, `*mut T` — 裸指针

## Sync：可以跨线程「共享引用」

**含义**：类型 `T: Sync` 表示 `&T` 可以安全地被多个线程同时访问。

换句话说：**T: Sync ⟺ &T: Send**

**是 Sync 的类型**：
- 不可变数据：i32, String, Vec 等
- Mutex<T>, RwLock<T>, Arc<T>
- 原子类型：AtomicBool, AtomicUsize

**不是 Sync 的类型**：
- `RefCell<T>`, `Cell<T>` — 非线程安全的内部可变性
- `Rc<T>`

## 对比

| Trait | 含义 | 问的问题 |
|-------|------|----------|
| Send | 所有权可跨线程转移 | 能 move 到新线程？ |
| Sync | 引用可被多线程共享 | 多线程同时读 &T？ |

## 编译器保护

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let rc = Rc::new(42);
    
    thread::spawn(move || {
        println!("{}", rc); // ❌ 编译错误！
    });
}
```

编译器报错：
```
error: `Rc<i32>` cannot be sent between threads safely
```

## 自动实现

Send 和 Sync 是 **auto trait**：
- 结构体所有字段都是 Send → 结构体也是 Send
- 结构体所有字段都是 Sync → 结构体也是 Sync

```rust
struct MyData {
    a: i32,      // Send + Sync ✅
    b: String,   // Send + Sync ✅
}
// MyData 自动是 Send + Sync

struct BadData {
    a: i32,
    b: Rc<i32>,  // 不是 Send ❌
}
// BadData 自动「不是」Send
```

---

**下节课**：原子类型 Atomics
