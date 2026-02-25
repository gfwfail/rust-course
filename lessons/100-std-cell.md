# 第 100 课：std::cell — 内部可变性的艺术

恭喜我们到达 100 课里程碑！今天讲一个 Rust 独特的核心概念：**内部可变性（Interior Mutability）**。

---

## 🤔 问题：借用规则的困境

Rust 的借用规则：
- 要么有多个不可变引用 `&T`
- 要么有一个可变引用 `&mut T`
- 不能同时存在

```rust
struct Counter {
    value: i32,
}

impl Counter {
    fn increment(&mut self) {
        self.value += 1;
    }
    
    fn get(&self) -> i32 {
        self.value
    }
}

let counter = Counter { value: 0 };
counter.increment();  // ❌ 编译错误！counter 不是 mut
```

但有时候，我们需要在「看起来不可变」的情况下修改数据：
- 缓存（懒加载）
- 统计访问次数
- 共享状态

---

## 📦 Cell<T> — 值类型的内部可变性

`Cell` 允许在持有 `&self` 时修改内部数据（通过复制）。

```rust
use std::cell::Cell;

struct Counter {
    value: Cell<i32>,
}

impl Counter {
    fn new() -> Self {
        Counter { value: Cell::new(0) }
    }
    
    // 注意：&self，不是 &mut self！
    fn increment(&self) {
        let v = self.value.get();
        self.value.set(v + 1);
    }
    
    fn get(&self) -> i32 {
        self.value.get()
    }
}

fn main() {
    let counter = Counter::new();  // 不需要 mut
    counter.increment();
    counter.increment();
    println!("Count: {}", counter.get());  // 2
}
```

### Cell 的限制

```rust
// Cell 只能用于 Copy 类型
let c: Cell<String> = Cell::new(String::from("hi"));
// c.get()  // ❌ String 不是 Copy，不能 get()

// 但可以用 take() 和 replace()
let old = c.take();      // 取出值，留下 Default
c.set(String::from("hello"));
let old = c.replace(String::from("world"));  // 换一个新的
```

---

## 📦 RefCell<T> — 运行时借用检查

`RefCell` 把编译期借用检查推迟到运行时，适用于非 Copy 类型。

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// 获取不可变引用
{
    let borrowed = data.borrow();  // 返回 Ref<Vec<i32>>
    println!("{:?}", *borrowed);   // [1, 2, 3]
}  // borrowed 离开作用域

// 获取可变引用
{
    let mut borrowed_mut = data.borrow_mut();  // 返回 RefMut<Vec<i32>>
    borrowed_mut.push(4);
}

println!("{:?}", data.borrow());  // [1, 2, 3, 4]
```

### ⚠️ 运行时 panic

```rust
let data = RefCell::new(42);

let r1 = data.borrow();      // 不可变借用
let r2 = data.borrow_mut();  // 💥 panic! 已经有不可变借用了

// 同样
let r1 = data.borrow_mut();  // 可变借用
let r2 = data.borrow_mut();  // 💥 panic! 已经有可变借用了
```

### 安全版本：try_borrow

```rust
let data = RefCell::new(42);
let r1 = data.borrow();

match data.try_borrow_mut() {
    Ok(mut r) => *r += 1,
    Err(_) => println!("无法获取可变引用"),
}
```

---

## 🔄 实战：在 &self 方法中修改状态

**场景：记录方法调用次数**

```rust
use std::cell::Cell;

struct Logger {
    message: String,
    call_count: Cell<u32>,  // 统计调用次数
}

impl Logger {
    fn new(msg: &str) -> Self {
        Logger {
            message: msg.to_string(),
            call_count: Cell::new(0),
        }
    }
    
    // 逻辑上是"只读"的，但内部更新计数器
    fn log(&self) {
        self.call_count.set(self.call_count.get() + 1);
        println!("[{}] {}", self.call_count.get(), self.message);
    }
}

fn main() {
    let logger = Logger::new("Hello, Rust!");
    logger.log();  // [1] Hello, Rust!
    logger.log();  // [2] Hello, Rust!
    logger.log();  // [3] Hello, Rust!
}
```

---

## 🧩 Cell vs RefCell 对比

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| 适用类型 | Copy 类型 | 任意类型 |
| 访问方式 | get/set（复制） | borrow/borrow_mut（引用） |
| 检查时机 | 无需检查 | 运行时检查 |
| panic 风险 | 无 | 违反借用规则会 panic |
| 性能 | 更快（无开销） | 有少量运行时开销 |

---

## 🔗 与 Rc 组合：共享可变状态

单线程中共享可变数据的经典模式：`Rc<RefCell<T>>`

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct SharedData {
    value: i32,
}

fn main() {
    // 创建共享的可变数据
    let data = Rc::new(RefCell::new(SharedData { value: 0 }));
    
    // 克隆引用（共享所有权）
    let data_clone = Rc::clone(&data);
    
    // 通过一个引用修改
    data.borrow_mut().value = 42;
    
    // 通过另一个引用读取
    println!("{:?}", data_clone.borrow());  // SharedData { value: 42 }
}
```

---

## 🧠 底层原理：UnsafeCell

所有内部可变性的基础是 `UnsafeCell<T>`，是唯一合法绕过不可变引用的方式。

```rust
use std::cell::UnsafeCell;

// Cell 和 RefCell 内部都用了 UnsafeCell
pub struct Cell<T> {
    value: UnsafeCell<T>,
}

// UnsafeCell 的核心方法
impl<T> UnsafeCell<T> {
    pub fn get(&self) -> *mut T {
        // 返回裸指针，允许修改
    }
}
```

⚠️ **普通代码不要直接用 UnsafeCell**，用 Cell/RefCell。

---

## 💡 什么时候用

| 场景 | 选择 |
|------|------|
| 简单计数器/标志位 | `Cell<i32>` / `Cell<bool>` |
| 缓存/懒初始化 | `Cell<Option<T>>` 或 `RefCell<Option<T>>` |
| 复杂数据结构的内部修改 | `RefCell<T>` |
| 多所有权 + 可变 | `Rc<RefCell<T>>` |
| 多线程 | `Arc<Mutex<T>>` (不要用 RefCell!) |

---

## 🎓 课后思考

1. 为什么 `Cell<T>` 要求 `T: Copy`？如果允许引用会怎样？
2. `RefCell` 为什么不能用于多线程？（提示：Sync trait）
3. 如何用 `RefCell` 实现一个懒加载的缓存？

---

*🎉 第 100 课完 — 感谢一路同行！*
