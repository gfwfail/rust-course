# 🎉 第 100 课：std::cell — 内部可变性

恭喜大家！我们到了第 100 课的里程碑！

今天讲一个 Rust 独特且重要的概念：**内部可变性 (Interior Mutability)**。

---

## 🤔 问题：Rust 的借用规则太严格？

Rust 的基本规则：
- 要么一个 `&mut T`（可变引用）
- 要么多个 `&T`（不可变引用）
- 不能同时存在

```rust
let mut x = 5;
let r1 = &x;
let r2 = &x;     // ✅ 多个不可变引用
// let rm = &mut x; // ❌ 已有不可变引用，不能再借可变的
```

这很安全，但有时太严格了。比如：
- 你想在多个地方共享数据，但偶尔需要修改
- 缓存：逻辑上是"读取"，但内部需要修改缓存
- 计数器：多处读取访问次数，但每次访问要 +1

---

## 🔓 内部可变性：绕过编译期检查

**内部可变性** 让你在只有 `&self`（不可变引用）的情况下修改数据。

核心思想：把借用检查从 **编译期** 移到 **运行期**。

标准库提供了几个工具：

| 类型 | 特点 | 线程安全 |
|------|------|----------|
| `Cell<T>` | 整体替换，T: Copy | ❌ |
| `RefCell<T>` | 运行期借用检查 | ❌ |
| `Mutex<T>` | 加锁 | ✅ |
| `RwLock<T>` | 读写锁 | ✅ |

今天聚焦单线程的 `Cell` 和 `RefCell`。

---

## 📦 Cell<T> — 简单值的内部可变性

`Cell` 适用于 `Copy` 类型，通过**整体替换**来修改：

```rust
use std::cell::Cell;

let cell = Cell::new(5);  // 注意：不需要 mut！

// 获取值（返回副本）
let val = cell.get();  // 5

// 设置新值
cell.set(10);
println!("{}", cell.get());  // 10

// 替换并返回旧值
let old = cell.replace(20);
println!("旧值: {}, 新值: {}", old, cell.get());  // 10, 20
```

### 为什么只能用于 Copy 类型？

因为 `get()` 返回的是**值的副本**，不是引用。如果 T 不能 Copy，就没法安全地"拿出来看看"。

```rust
// ❌ String 不是 Copy，不能用 Cell
// let cell = Cell::new(String::from("hello"));
// let s = cell.get();  // 编译错误！

// ✅ 对于非 Copy 类型，用 RefCell
```

### 实际例子：访问计数器

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<u32>,
}

impl Counter {
    fn new() -> Self {
        Counter { count: Cell::new(0) }
    }
    
    // 注意：&self 不是 &mut self！
    fn increment(&self) {
        self.count.set(self.count.get() + 1);
    }
    
    fn get(&self) -> u32 {
        self.count.get()
    }
}

fn main() {
    let counter = Counter::new();  // 不需要 mut
    counter.increment();
    counter.increment();
    println!("计数: {}", counter.get());  // 2
}
```

---

## 🔄 RefCell<T> — 运行期借用检查

`RefCell` 适用于任何类型，通过**运行期借用检查**实现可变性：

```rust
use std::cell::RefCell;

let cell = RefCell::new(String::from("hello"));

// 不可变借用
{
    let s = cell.borrow();  // 返回 Ref<String>
    println!("{}", *s);      // hello
}  // s 在这里被 drop，借用结束

// 可变借用
{
    let mut s = cell.borrow_mut();  // 返回 RefMut<String>
    s.push_str(" world");
}

println!("{:?}", cell.borrow());  // "hello world"
```

### ⚠️ 运行期 panic！

借用规则仍然存在，只是在运行期检查：

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);

let r1 = cell.borrow();      // 不可变借用
let r2 = cell.borrow();      // ✅ 可以多个不可变借用
// let rm = cell.borrow_mut(); // 💥 panic! 已有不可变借用

drop(r1);
drop(r2);
let rm = cell.borrow_mut();  // ✅ 之前的借用都结束了
```

### 安全版本：try_borrow / try_borrow_mut

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);
let r = cell.borrow();

// 不会 panic，返回 Result
match cell.try_borrow_mut() {
    Ok(mut rm) => *rm = 10,
    Err(_) => println!("借用冲突！"),
}
```

---

## 🧩 经典组合：Rc<RefCell<T>>

单独的 `RefCell` 只有一个所有者。想要多个所有者共享可变数据？

```rust
use std::rc::Rc;
use std::cell::RefCell;

// 多个所有者，每个都能修改
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

let a = Rc::clone(&shared);
let b = Rc::clone(&shared);

// a 和 b 都可以修改同一个 Vec
a.borrow_mut().push(4);
b.borrow_mut().push(5);

println!("{:?}", shared.borrow());  // [1, 2, 3, 4, 5]
```

### 实际例子：双向链表节点

```rust
use std::rc::Rc;
use std::cell::RefCell;

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    value: T,
    next: Link<T>,
    prev: Link<T>,  // 需要 Rc + RefCell 才能实现
}
```

---

## 🆚 Cell vs RefCell

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| T 的要求 | T: Copy | 任意 T |
| 访问方式 | get/set（复制） | borrow/borrow_mut（引用） |
| 性能 | 更快（无运行期检查） | 稍慢（维护借用计数） |
| panic 风险 | 无 | borrow 冲突会 panic |
| 适用场景 | 简单值（i32, bool...） | 复杂类型（String, Vec...） |

---

## 💡 什么时候用内部可变性？

**适合用的场景：**
1. 缓存/记忆化
2. 访问计数器、统计信息
3. 需要共享所有权且可变（Rc<RefCell<T>>）
4. Mock 对象（测试时需要记录调用）

**不适合的场景：**
1. 能用 `&mut self` 就用 `&mut self`
2. 多线程（用 Mutex/RwLock 代替）
3. 性能敏感的热路径

---

## 🎓 课后思考

1. 为什么 `Cell` 和 `RefCell` 都不能跨线程（`!Sync`）？
2. 如果 `RefCell::borrow_mut()` 可能 panic，为什么还要用它而不直接用 `&mut`？
3. `Rc<RefCell<T>>` 和 `Arc<Mutex<T>>` 有什么区别？

---

*🎉 第 100 课完！感谢大家陪伴走过 100 节课！*
