# 🎉 第 100 课：std::cell — 内部可变性

恭喜！我们到了第 100 课！今天讲 Rust 中一个重要概念：**内部可变性 (Interior Mutability)**。

---

## 🤔 问题：借用规则的困境

Rust 的借用规则很严格：
- 要么一个可变引用 `&mut T`
- 要么多个不可变引用 `&T`
- 不能同时存在

```rust
let mut x = 5;
let r1 = &x;
let r2 = &x;
let r3 = &mut x;  // ❌ 编译错误！已有不可变借用
```

**但有时我们需要通过不可变引用修改数据！**

---

## 📦 Cell<T> — 最简单的内部可变性

`Cell` 通过 **复制** 值来实现内部可变性：

```rust
use std::cell::Cell;

let x = Cell::new(5);  // 注意：x 不是 mut！

x.set(10);             // 通过不可变引用修改！
println!("{}", x.get()); // 10
```

### 为什么能绕过借用检查？

```rust
// Cell 内部大致是这样
pub struct Cell<T> {
    value: UnsafeCell<T>,  // 魔法在这里
}

impl<T: Copy> Cell<T> {
    pub fn get(&self) -> T {   // 注意：&self，不是 &mut self
        // 返回值的副本
    }
    
    pub fn set(&self, val: T) { // 还是 &self！
        // 写入新值
    }
}
```

`Cell` 的关键限制：**T 必须实现 Copy**

```rust
let name = Cell::new(String::from("hello"));  // ❌ String 不是 Copy
```

---

## 🔓 RefCell<T> — 运行时借用检查

如果 T 不是 Copy（比如 String、Vec），用 `RefCell`：

```rust
use std::cell::RefCell;

let name = RefCell::new(String::from("hello"));

// borrow() 返回 Ref<T>，类似 &T
println!("{}", name.borrow());  // hello

// borrow_mut() 返回 RefMut<T>，类似 &mut T
name.borrow_mut().push_str(" world");
println!("{}", name.borrow());  // hello world
```

### 编译时 vs 运行时检查

```rust
// 普通引用：编译时检查
let mut s = String::new();
let r1 = &s;
let r2 = &mut s;  // ❌ 编译错误

// RefCell：运行时检查
let s = RefCell::new(String::new());
let r1 = s.borrow();      // Ok
let r2 = s.borrow_mut();  // 💥 panic! 运行时错误
```

**RefCell 在运行时追踪借用状态，违规时 panic！**

---

## 🎯 实际应用：在结构体中共享可变状态

```rust
use std::cell::RefCell;
use std::rc::Rc;

// 想象一个计数器，被多个地方共享
struct Counter {
    value: RefCell<i32>,
}

impl Counter {
    fn new() -> Self {
        Counter { value: RefCell::new(0) }
    }
    
    fn increment(&self) {  // 注意：&self，不是 &mut self
        *self.value.borrow_mut() += 1;
    }
    
    fn get(&self) -> i32 {
        *self.value.borrow()
    }
}

fn main() {
    let counter = Rc::new(Counter::new());
    
    // 多个地方持有同一个 counter
    let c1 = Rc::clone(&counter);
    let c2 = Rc::clone(&counter);
    
    c1.increment();
    c2.increment();
    
    println!("{}", counter.get());  // 2
}
```

**PHP 类比**：这就像 PHP 的普通对象行为 —— 多处持有同一个对象引用，都能修改。但 Rust 默认不允许，需要显式用 `RefCell`。

---

## ⚡ Cell vs RefCell 对比

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| T 的要求 | 必须 Copy | 无限制 |
| 访问方式 | get/set 复制值 | borrow/borrow_mut |
| 性能 | 更快（无运行时检查） | 有少量开销 |
| 安全性 | 编译时保证 | 可能 panic |

**选择原则**：
- T 是 Copy（i32, bool, char 等）→ 用 `Cell`
- T 不是 Copy（String, Vec 等）→ 用 `RefCell`

---

## 🔒 OnceCell — 只写一次

```rust
use std::cell::OnceCell;

let cell: OnceCell<String> = OnceCell::new();

// 第一次写入成功
cell.set(String::from("hello")).unwrap();

// 第二次写入失败（返回 Err）
assert!(cell.set(String::from("world")).is_err());

// 读取
println!("{}", cell.get().unwrap());  // hello
```

**用途**：懒初始化、全局配置

```rust
use std::cell::OnceCell;

fn get_config() -> &'static str {
    static CONFIG: OnceCell<String> = OnceCell::new();
    CONFIG.get_or_init(|| {
        // 只在第一次调用时执行
        std::fs::read_to_string("config.txt").unwrap()
    })
}
```

---

## 🧵 线程安全版本

`Cell` 和 `RefCell` **不是**线程安全的！

多线程环境用：
- `Cell` → `std::sync::atomic` (AtomicI32 等)
- `RefCell` → `std::sync::Mutex` 或 `RwLock`
- `OnceCell` → `std::sync::OnceLock`

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));

// 多线程安全地修改
let c = Arc::clone(&counter);
std::thread::spawn(move || {
    *c.lock().unwrap() += 1;
});
```

我们下节课详细讲 Mutex！

---

## 💡 总结

```
                  ┌─────────────────┐
                  │  需要内部可变性？ │
                  └────────┬────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
         T: Copy?                   T: !Copy?
              │                         │
              ▼                         ▼
         Cell<T>                   RefCell<T>
       (复制值)                  (运行时借用检查)
```

**记住**：
- 内部可变性 = 通过 `&self` 修改数据
- `Cell`：值复制，零开销
- `RefCell`：运行时借用检查，违规 panic
- 多线程用 `Mutex`/`RwLock`

---

## 🎓 课后思考

1. 为什么 `Cell<String>` 编译不过？
2. `RefCell` 什么时候会 panic？
3. 为什么说 `Cell`/`RefCell` 不是线程安全的？

---

*🎉 第 100 课完 —— 恭喜坚持到这里！*
