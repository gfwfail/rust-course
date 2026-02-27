# 第 100 课：std::cell — 内部可变性 (Interior Mutability)

恭喜各位坚持到第 100 课！今天讲一个 Rust 的核心概念：**内部可变性**。

---

## 🤔 问题：为什么需要内部可变性？

Rust 的借用规则很严格：

```rust
// 要么一个 &mut（可变借用）
// 要么任意多个 &（不可变借用）
// 不能同时存在！
```

但有时候，我们需要在只有 `&self`（不可变引用）的情况下修改内部数据：

```rust
struct Counter {
    count: i32,
}

impl Counter {
    // 编译错误！&self 不能修改 count
    fn increment(&self) {
        self.count += 1;  // ❌ cannot borrow as mutable
    }
}
```

这时候就需要 **内部可变性** —— 在编译时看起来不可变，但运行时可以安全地修改。

---

## 📦 Cell<T> — 简单值的内部可变性

`Cell<T>` 适用于 `Copy` 类型（整数、布尔等）：

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<i32>,  // 用 Cell 包装
}

impl Counter {
    fn new() -> Self {
        Counter { count: Cell::new(0) }
    }
    
    // 现在可以用 &self 修改了！
    fn increment(&self) {
        let current = self.count.get();  // 取出值
        self.count.set(current + 1);     // 设置新值
    }
    
    fn get(&self) -> i32 {
        self.count.get()
    }
}

fn main() {
    let counter = Counter::new();
    counter.increment();
    counter.increment();
    println!("Count: {}", counter.get());  // Count: 2
}
```

### Cell 的特点

- ✅ 零运行时开销
- ✅ 通过 `get()` / `set()` 操作（值的复制）
- ❌ 只能用于 `Copy` 类型
- ❌ 不能获取内部的引用

---

## 📦 RefCell<T> — 任意类型的内部可变性

`RefCell<T>` 适用于任何类型，通过 **运行时借用检查**：

```rust
use std::cell::RefCell;

struct Cache {
    data: RefCell<Vec<String>>,
}

impl Cache {
    fn new() -> Self {
        Cache { data: RefCell::new(Vec::new()) }
    }
    
    // &self 但可以修改内部 Vec
    fn add(&self, item: String) {
        self.data.borrow_mut().push(item);
    }
    
    fn get_all(&self) -> Vec<String> {
        self.data.borrow().clone()
    }
}
```

### 借用方法

```rust
use std::cell::RefCell;

let cell = RefCell::new(vec![1, 2, 3]);

// 不可变借用（类似 &T）
let borrowed = cell.borrow();
println!("{:?}", *borrowed);

// 可变借用（类似 &mut T）
let mut borrowed_mut = cell.borrow_mut();
borrowed_mut.push(4);
```

---

## ⚠️ RefCell 的运行时检查

借用规则在运行时检查，违反会 **panic**：

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);

let r1 = cell.borrow();      // 不可变借用
let r2 = cell.borrow();      // ✅ 可以有多个不可变借用
let r3 = cell.borrow_mut();  // 💥 panic! 已有不可变借用

// 同理
let m1 = cell.borrow_mut();  // 可变借用
let m2 = cell.borrow_mut();  // 💥 panic! 已有可变借用
```

### 安全的写法：try_borrow

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);

// 用 try_borrow 避免 panic
if let Ok(borrowed) = cell.try_borrow() {
    println!("借用成功: {}", *borrowed);
}

if let Ok(mut borrowed) = cell.try_borrow_mut() {
    *borrowed += 1;
}
```

---

## 🧩 实战：缓存计算结果

经典场景 —— 惰性计算 + 缓存：

```rust
use std::cell::RefCell;

struct LazyValue {
    computation: fn() -> i32,
    cache: RefCell<Option<i32>>,
}

impl LazyValue {
    fn new(computation: fn() -> i32) -> Self {
        LazyValue {
            computation,
            cache: RefCell::new(None),
        }
    }
    
    // &self 但可以缓存结果
    fn get(&self) -> i32 {
        // 如果已缓存，直接返回
        if let Some(value) = *self.cache.borrow() {
            return value;
        }
        
        // 否则计算并缓存
        let value = (self.computation)();
        *self.cache.borrow_mut() = Some(value);
        value
    }
}

fn expensive() -> i32 {
    println!("计算中...");
    42
}

fn main() {
    let lazy = LazyValue::new(expensive);
    
    println!("{}", lazy.get());  // 计算中... 42
    println!("{}", lazy.get());  // 42 (直接用缓存)
}
```

---

## 📊 Cell vs RefCell

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| 适用类型 | 仅 `Copy` | 任意类型 |
| 操作方式 | get/set (复制) | borrow/borrow_mut (引用) |
| 运行时开销 | 无 | 借用计数 |
| 违反规则 | 编译错误 | panic |
| 线程安全 | ❌ | ❌ |

---

## 🔒 线程安全版本

`Cell` 和 `RefCell` 都 **不是** 线程安全的！

多线程场景用：
- `Mutex<T>` — 对应 `RefCell`
- `RwLock<T>` — 读写锁
- `AtomicXxx` — 对应 `Cell`（原子类型）

```rust
// 单线程
use std::cell::RefCell;
let data = RefCell::new(vec![]);

// 多线程
use std::sync::Mutex;
let data = Mutex::new(vec![]);
```

---

## 🎓 课后思考

1. 什么场景适合用 `Cell`？什么场景用 `RefCell`？
2. `RefCell::borrow_mut()` 为什么会 panic 而不是编译错误？
3. 如何实现一个线程安全的惰性值（结合 `Mutex` 或 `OnceCell`）？

---

*🎉 第 100 课完！感谢各位坚持学习！*
