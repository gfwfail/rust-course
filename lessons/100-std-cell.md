# 第 100 课：std::cell — 内部可变性

恭喜来到第 100 课！今天讲一个 Rust 独特而重要的概念：**内部可变性 (Interior Mutability)**。

---

## 🤔 问题：借用规则的困境

Rust 的借用规则很严格：

```rust
// 要么一个可变引用
let mut x = 5;
let r = &mut x;

// 要么多个不可变引用
let r1 = &x;
let r2 = &x;

// 但不能同时有！
```

这在大多数情况下没问题，但有时候我们需要在"看起来不可变"的东西里面修改数据：

```rust
struct Counter {
    count: u32,
}

impl Counter {
    // ❌ 编译错误！&self 是不可变引用
    fn increment(&self) {
        self.count += 1;  // 不能修改
    }
}
```

---

## 📦 Cell<T> — 值的内部可变性

`Cell<T>` 适用于 `Copy` 类型，通过**整体替换**来修改：

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<u32>,
}

impl Counter {
    fn new() -> Self {
        Counter { count: Cell::new(0) }
    }
    
    // ✅ &self 也能修改！
    fn increment(&self) {
        let current = self.count.get();
        self.count.set(current + 1);
    }
    
    fn get(&self) -> u32 {
        self.count.get()
    }
}

fn main() {
    let counter = Counter::new();  // 不需要 mut
    counter.increment();
    counter.increment();
    println!("{}", counter.get());  // 2
}
```

### Cell 的 API

```rust
use std::cell::Cell;

let c = Cell::new(5);

// get() — 返回值的拷贝
let val = c.get();  // 5

// set() — 替换整个值
c.set(10);

// replace() — 替换并返回旧值
let old = c.replace(20);  // old = 10

// take() — 取出值，留下默认值（需要 T: Default）
let c = Cell::new(5);
let val = c.take();  // val = 5, c 现在是 0
```

**⚠️ 限制**：`Cell<T>` 要求 `T: Copy`（对于 `get()` 方法），因为 `get()` 返回拷贝。

---

## 📦 RefCell<T> — 运行时借用检查

`RefCell<T>` 可以用于任何类型，借用检查从编译时移到运行时：

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// borrow() — 获取不可变引用
{
    let r = data.borrow();
    println!("{:?}", *r);  // [1, 2, 3]
}  // r 离开作用域，借用结束

// borrow_mut() — 获取可变引用
{
    let mut w = data.borrow_mut();
    w.push(4);
}

println!("{:?}", data.borrow());  // [1, 2, 3, 4]
```

### 运行时 panic！

```rust
use std::cell::RefCell;

let data = RefCell::new(5);

let r1 = data.borrow();      // 不可变借用
let r2 = data.borrow();      // ✅ 可以多个不可变借用

// ❌ panic! 运行时崩溃
let w = data.borrow_mut();   // 已有不可变借用，不能可变借用
```

### 安全版本：try_borrow

```rust
use std::cell::RefCell;

let data = RefCell::new(5);
let r = data.borrow();

// 不会 panic，返回 Result
match data.try_borrow_mut() {
    Ok(mut w) => *w = 10,
    Err(_) => println!("借用失败"),
}
```

---

## 🆚 Cell vs RefCell

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| 适用类型 | T: Copy | 任意 T |
| 获取方式 | 拷贝值 | 借用引用 |
| 性能 | 无开销 | 运行时检查 |
| 违规后果 | 编译错误 | 运行时 panic |

**选择原则**：
- 简单的 `Copy` 类型（数字、bool）→ `Cell`
- 其他类型 → `RefCell`

---

## 💡 实际应用：缓存（Memoization）

```rust
use std::cell::RefCell;

struct CachedComputer {
    value: RefCell<Option<i32>>,
}

impl CachedComputer {
    fn new() -> Self {
        CachedComputer {
            value: RefCell::new(None),
        }
    }
    
    // &self 但能缓存结果
    fn compute(&self) -> i32 {
        // 先检查缓存
        if let Some(v) = *self.value.borrow() {
            return v;
        }
        
        // 计算（假设很耗时）
        let result = 42;
        
        // 存入缓存
        *self.value.borrow_mut() = Some(result);
        
        result
    }
}
```

---

## 💡 实际应用：观察者模式

```rust
use std::cell::RefCell;

type Callback = Box<dyn Fn(i32)>;

struct Observable {
    value: i32,
    observers: RefCell<Vec<Callback>>,
}

impl Observable {
    fn new(value: i32) -> Self {
        Observable {
            value,
            observers: RefCell::new(Vec::new()),
        }
    }
    
    // &self 也能添加观察者
    fn add_observer(&self, callback: Callback) {
        self.observers.borrow_mut().push(callback);
    }
    
    fn notify(&self) {
        for callback in self.observers.borrow().iter() {
            callback(self.value);
        }
    }
}
```

---

## 🔗 与 Rc 配合使用

`RefCell` 常与 `Rc`（引用计数）配合，实现多所有者可变数据：

```rust
use std::cell::RefCell;
use std::rc::Rc;

let shared_data = Rc::new(RefCell::new(vec![1, 2, 3]));

let clone1 = Rc::clone(&shared_data);
let clone2 = Rc::clone(&shared_data);

// 两个所有者都能修改！
clone1.borrow_mut().push(4);
clone2.borrow_mut().push(5);

println!("{:?}", shared_data.borrow());  // [1, 2, 3, 4, 5]
```

这是 Rust 中实现共享可变状态的常见模式。

---

## ⚠️ Cell 和 RefCell 不是线程安全的！

```rust
use std::cell::RefCell;
use std::thread;

let data = RefCell::new(5);

// ❌ 编译错误！RefCell 不实现 Sync
thread::spawn(move || {
    *data.borrow_mut() = 10;
});
```

`Cell` 和 `RefCell` 都没有实现 `Sync` trait，不能在线程间共享。

多线程场景使用：
- `Mutex<T>` — 互斥锁
- `RwLock<T>` — 读写锁
- `AtomicXxx` — 原子类型

---

## 🧠 内部可变性的本质

```
编译时借用检查（默认）
     ↓ 某些场景不够灵活
运行时借用检查（Cell/RefCell）
     ↓ 多线程
Mutex / RwLock（加锁保护）
```

**记住**：内部可变性是"逃生舱"，优先使用编译时检查。只有在确实需要时才使用 `Cell`/`RefCell`。

---

## 📝 总结

| 类型 | 用途 | 线程安全 |
|------|------|----------|
| Cell<T> | Copy 类型内部可变 | ❌ |
| RefCell<T> | 任意类型内部可变 | ❌ |
| Mutex<T> | 多线程内部可变 | ✅ |
| RwLock<T> | 多线程读写锁 | ✅ |

---

## 🎓 课后思考

1. 为什么 `Cell<T>` 只支持 `Copy` 类型？（提示：考虑 `get()` 如果返回引用会怎样）
2. `RefCell` 的运行时开销是什么？（提示：需要跟踪借用计数）
3. 什么时候该用 `Cell`/`RefCell`，什么时候该重构代码结构避免内部可变性？

---

*🎉 恭喜完成第 100 课！我们已经系统学习了 Rust 标准库的核心模块！*
