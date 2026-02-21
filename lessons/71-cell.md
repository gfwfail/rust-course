# 第 71 课：Cell — 轻量级内部可变性

> 日期：2026-02-21  
> 主题：Cell 类型与内部可变性

---

## 📍 RefCell vs Cell

```
RefCell<T>  →  运行时借用检查，可以获取 &T 和 &mut T
Cell<T>     →  无借用检查，只能整体 get/set
```

**核心区别：**
- `RefCell` 允许你借用内部值（`borrow()` / `borrow_mut()`）
- `Cell` 不允许借用，只能 **拷贝出来** 或 **替换进去**

---

## 🔧 Cell 基础用法

```rust
use std::cell::Cell;

fn main() {
    let counter = Cell::new(0);
    
    // get: 拷贝出值（需要 T: Copy）
    println!("当前值: {}", counter.get());
    
    // set: 替换整个值
    counter.set(42);
    println!("设置后: {}", counter.get());
    
    // 在不可变引用上修改！
    increment(&counter);
    println!("递增后: {}", counter.get());
}

fn increment(c: &Cell<i32>) {
    c.set(c.get() + 1);
}
```

**注意：** `get()` 需要 `T: Copy`，因为它是拷贝语义。

---

## 💡 为什么需要 Cell？

想象一个场景：你有一个结构体，大部分字段不变，但有个计数器需要改：

```rust
use std::cell::Cell;

struct CachedComputation {
    value: String,           // 不变
    access_count: Cell<u32>, // 需要变
}

impl CachedComputation {
    fn get_value(&self) -> &str {
        // 虽然是 &self，但能修改 access_count！
        self.access_count.set(self.access_count.get() + 1);
        &self.value
    }
    
    fn times_accessed(&self) -> u32 {
        self.access_count.get()
    }
}

fn main() {
    let cache = CachedComputation {
        value: String::from("expensive result"),
        access_count: Cell::new(0),
    };
    
    println!("{}", cache.get_value());
    println!("{}", cache.get_value());
    println!("访问次数: {}", cache.times_accessed()); // 2
}
```

用 PHP/Laravel 类比：
```php
// PHP 里你可以随便改
class CachedComputation {
    private int $accessCount = 0;
    
    public function getValue(): string {
        $this->accessCount++;  // 直接改
        return $this->value;
    }
}
```

但 Rust 的 `&self` 意味着「不可变借用」，正常情况下不能改任何字段。
`Cell` 打破了这个限制 —— 合法地在 `&self` 里修改！

---

## 🎯 Cell vs RefCell 选择指南

| 特性 | Cell | RefCell |
|------|------|---------|
| 内部值要求 | `T: Copy` | 任意 `T` |
| 借用内部值 | ❌ 不能 | ✅ 可以 |
| 运行时开销 | 零 | 有（借用计数） |
| panic 风险 | 无 | 有（借用冲突） |
| 适用场景 | 简单类型（i32, bool） | 复杂类型（Vec, String） |

**经验法则：**
- 能用 `Cell` 就用 `Cell`（更简单、更安全）
- 需要借用内部值时才用 `RefCell`

---

## 🔥 实战：共享可变状态

```rust
use std::cell::Cell;

// 一个简单的 ID 生成器
struct IdGenerator {
    next_id: Cell<u64>,
}

impl IdGenerator {
    fn new() -> Self {
        IdGenerator { next_id: Cell::new(1) }
    }
    
    // &self 但能生成递增 ID！
    fn next(&self) -> u64 {
        let id = self.next_id.get();
        self.next_id.set(id + 1);
        id
    }
}

fn main() {
    let gen = IdGenerator::new();
    
    println!("ID: {}", gen.next()); // 1
    println!("ID: {}", gen.next()); // 2
    println!("ID: {}", gen.next()); // 3
}
```

---

## 📦 Cell 的其他方法

```rust
use std::cell::Cell;

fn main() {
    let cell = Cell::new(10);
    
    // replace: 设置新值，返回旧值
    let old = cell.replace(20);
    println!("旧值: {}, 新值: {}", old, cell.get());
    
    // take: 取出值，留下默认值（需要 T: Default）
    let cell = Cell::new(String::from("hello"));
    let s = cell.take();  // cell 变成 ""
    println!("取出: {}", s);
    
    // swap: 交换两个 Cell 的值
    let a = Cell::new(1);
    let b = Cell::new(2);
    a.swap(&b);
    println!("a={}, b={}", a.get(), b.get()); // a=2, b=1
    
    // update: 用闭包更新值（需要 T: Copy）
    let cell = Cell::new(10);
    cell.update(|x| x * 2);
    println!("更新后: {}", cell.get()); // 20
}
```

---

## ⚡ 为什么 Cell 是零开销？

`Cell` 的实现极其简单：

```rust
// 简化版实现（标准库实际实现）
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}

impl<T: Copy> Cell<T> {
    pub fn get(&self) -> T {
        // SAFETY: 我们只返回拷贝，不返回引用
        unsafe { *self.value.get() }
    }
    
    pub fn set(&self, val: T) {
        // SAFETY: 没有引用指向内部值
        unsafe { *self.value.get() = val; }
    }
}
```

底层用 `UnsafeCell`（Rust 内部可变性的根基），但 API 完全安全。

**为什么安全？**
- `get()` 返回拷贝，不是引用 → 没有悬垂引用风险
- `set()` 直接覆盖 → 没有借用冲突
- 单线程（`Cell` 不是 `Sync`）→ 没有数据竞争

---

## 🚫 Cell 不是线程安全的

```rust
use std::cell::Cell;

fn main() {
    let cell = Cell::new(0);
    
    // 编译错误！Cell 不是 Sync
    // std::thread::spawn(|| {
    //     cell.set(1);  // ❌ Cell<i32> cannot be shared between threads safely
    // });
}
```

**多线程场景用什么？**
- `AtomicI32` / `AtomicU64` 等原子类型
- `Mutex<T>` / `RwLock<T>`
- `Arc<Atomic*>` 组合

---

## 🎭 Cell 使用模式

### 模式 1：缓存计数

```rust
use std::cell::Cell;

struct Parser {
    line_count: Cell<usize>,
}

impl Parser {
    fn parse(&self, input: &str) {
        for _ in input.lines() {
            self.line_count.set(self.line_count.get() + 1);
        }
    }
}
```

### 模式 2：惰性标记

```rust
use std::cell::Cell;

struct Connection {
    is_closed: Cell<bool>,
}

impl Connection {
    fn close(&self) {
        if !self.is_closed.get() {
            // 执行关闭逻辑...
            self.is_closed.set(true);
        }
    }
}
```

### 模式 3：递归深度限制

```rust
use std::cell::Cell;

struct Visitor {
    depth: Cell<u32>,
    max_depth: u32,
}

impl Visitor {
    fn visit(&self, node: &str) {
        if self.depth.get() >= self.max_depth {
            println!("达到最大深度，停止");
            return;
        }
        
        self.depth.set(self.depth.get() + 1);
        println!("访问 {} (深度 {})", node, self.depth.get());
        // ... 递归访问子节点 ...
        self.depth.set(self.depth.get() - 1);
    }
}
```

---

## 🧠 本课要点

1. **Cell 用于 Copy 类型的内部可变性**
2. **不能借用内部值**，只能 get/set/replace/take
3. **零运行时开销**，比 RefCell 更轻量
4. **永远不会 panic**（不像 RefCell 的借用检查）
5. **不是线程安全的**（不是 `Sync`）
6. **典型用途**：计数器、标志位、简单状态

---

## 📝 练习思考

1. 为什么 `Cell<Vec<T>>` 不实用？
   - 答案：`Vec<T>` 不是 `Copy`，所以 `get()` 不能用。只能用 `take()` 把整个 Vec 拿出来。

2. 如果需要在 `&self` 里修改 `HashMap`，该用什么？
   - 答案：`RefCell<HashMap<K, V>>`，因为 HashMap 不是 Copy，而且你需要调用它的方法（需要借用）

3. `Cell` 是线程安全的吗？为什么？
   - 答案：不是。`Cell` 没有实现 `Sync`，因为它的 get/set 不是原子操作，在多线程下会有数据竞争。

---

## 📚 相关文档

- [std::cell::Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html)
- [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html)
- [Interior Mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)

---

*下节课预告：Cow — 写时克隆的智能指针*
