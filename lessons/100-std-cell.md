# 第 100 课：std::cell — 内部可变性 🎉

恭喜来到第 100 课！今天讲一个 Rust 独特且重要的概念：**内部可变性（Interior Mutability）**。

---

## 🤔 问题的起源

Rust 的借用规则：要么多个不可变引用，要么一个可变引用。

```rust
let mut x = 5;
let r1 = &x;     // 不可变引用
let r2 = &x;     // OK，可以多个
// let r3 = &mut x;  // ❌ 编译错误！已有不可变引用

// PHP/JS 可没这限制...
```

但有时候，我们需要在**看起来不可变**的情况下修改数据：

```rust
// 比如想实现一个带缓存的结构
struct CachedValue {
    value: i32,
    cached: Option<i32>,  // 想在 get() 时惰性计算并缓存
}

impl CachedValue {
    fn get(&self) -> i32 {  // &self 是不可变引用
        if self.cached.is_none() {
            // self.cached = Some(expensive_compute(self.value));
            // ❌ 不能修改！self 是 &self
        }
        self.cached.unwrap()
    }
}
```

这就是 **内部可变性** 要解决的问题。

---

## 📦 Cell<T> — 单线程值拷贝

`Cell<T>` 是最简单的内部可变性容器，适用于 `Copy` 类型：

```rust
use std::cell::Cell;

let cell = Cell::new(5);  // 注意：不需要 mut！

// 读取：拷贝出来
let value = cell.get();   // 5

// 写入：整个替换
cell.set(10);
println!("{}", cell.get());  // 10

// 不能获取引用，只能 get/set
// let r = &cell.get();  // 这是 copy 出来的值的引用
```

### Cell 的特点

1. **不需要 `mut`** 就能修改
2. **只支持 `Copy` 类型**（因为 get 会拷贝）
3. **没有运行时开销**（零成本抽象）
4. **单线程**（不是 Sync）

```rust
// 实际用例：计数器
struct Counter {
    count: Cell<i32>,
}

impl Counter {
    fn new() -> Self {
        Counter { count: Cell::new(0) }
    }
    
    fn increment(&self) {  // &self，不需要 &mut self
        self.count.set(self.count.get() + 1);
    }
    
    fn get(&self) -> i32 {
        self.count.get()
    }
}

let counter = Counter::new();  // 不需要 mut
counter.increment();
counter.increment();
println!("{}", counter.get());  // 2
```

---

## 📦 RefCell<T> — 单线程借用检查

`RefCell<T>` 把借用检查从编译期推迟到运行时：

```rust
use std::cell::RefCell;

let cell = RefCell::new(vec![1, 2, 3]);

// 借用：返回 Ref<T>（类似 &T）
{
    let borrowed = cell.borrow();  // 不可变借用
    println!("{:?}", *borrowed);   // [1, 2, 3]
}  // borrowed 离开作用域，释放借用

// 可变借用：返回 RefMut<T>（类似 &mut T）
{
    let mut borrowed = cell.borrow_mut();  // 可变借用
    borrowed.push(4);
}

println!("{:?}", cell.borrow());  // [1, 2, 3, 4]
```

### ⚠️ 运行时检查

借用规则在运行时检查，违反会 **panic**：

```rust
let cell = RefCell::new(5);

let r1 = cell.borrow();      // 不可变借用
let r2 = cell.borrow();      // OK，多个不可变借用

// let r3 = cell.borrow_mut();  
// ❌ panic! 已有不可变借用时不能可变借用
```

```rust
let cell = RefCell::new(5);

let r1 = cell.borrow_mut();   // 可变借用
// let r2 = cell.borrow_mut(); 
// ❌ panic! 只能有一个可变借用
```

### try_borrow — 不 panic 的版本

```rust
let cell = RefCell::new(5);
let r1 = cell.borrow();

// 用 try_borrow_mut 返回 Result
match cell.try_borrow_mut() {
    Ok(mut r) => *r = 10,
    Err(_) => println!("借用冲突！"),
}
```

---

## 🎯 实战：带缓存的惰性计算

```rust
use std::cell::RefCell;

struct LazyCache<T, F>
where
    F: Fn() -> T,
{
    compute: F,
    cache: RefCell<Option<T>>,
}

impl<T, F> LazyCache<T, F>
where
    T: Clone,
    F: Fn() -> T,
{
    fn new(compute: F) -> Self {
        LazyCache {
            compute,
            cache: RefCell::new(None),
        }
    }

    fn get(&self) -> T {  // &self，不是 &mut self！
        // 检查缓存
        if self.cache.borrow().is_none() {
            // 计算并缓存
            let value = (self.compute)();
            *self.cache.borrow_mut() = Some(value);
        }
        self.cache.borrow().clone().unwrap()
    }
}

// 使用
let expensive = LazyCache::new(|| {
    println!("Computing...");
    42
});

println!("{}", expensive.get());  // Computing... 42
println!("{}", expensive.get());  // 42（没有重新计算）
```

---

## 🌳 Rc<RefCell<T>> — 共享可变状态

`Rc` 提供共享所有权，`RefCell` 提供内部可变性，组合起来：

```rust
use std::rc::Rc;
use std::cell::RefCell;

// 多个所有者共享可变数据
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

let a = Rc::clone(&shared);
let b = Rc::clone(&shared);

// a 和 b 都可以修改
a.borrow_mut().push(4);
b.borrow_mut().push(5);

println!("{:?}", shared.borrow());  // [1, 2, 3, 4, 5]
```

### 经典用例：双向链表 / 图结构

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
}

let node1 = Rc::new(RefCell::new(Node {
    value: 1,
    next: None,
}));

let node2 = Rc::new(RefCell::new(Node {
    value: 2,
    next: Some(Rc::clone(&node1)),
}));

// 修改 node1
node1.borrow_mut().value = 100;

// 通过 node2 访问 node1
let next = node2.borrow().next.as_ref().unwrap().borrow().value;
println!("node2.next.value = {}", next);  // 100
```

---

## 📊 Cell vs RefCell

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| T 的要求 | Copy | 无 |
| 访问方式 | get/set | borrow/borrow_mut |
| 能获取引用？ | ❌ | ✅ |
| 运行时开销 | 零 | 极小（引用计数） |
| 违反借用规则 | 不可能 | panic |

**选择原则**：
- `Copy` 类型（i32, bool, char...）→ `Cell`
- 其他类型 → `RefCell`

---

## 🔐 OnceCell — 一次性初始化

```rust
use std::cell::OnceCell;

let cell = OnceCell::new();

// 只能设置一次
assert!(cell.set(42).is_ok());
assert!(cell.set(100).is_err());  // 第二次设置失败

// 获取值
assert_eq!(cell.get(), Some(&42));

// get_or_init：惰性初始化
let cell: OnceCell<String> = OnceCell::new();
let value = cell.get_or_init(|| {
    println!("Initializing...");
    "hello".to_string()
});
println!("{}", value);  // "hello"

// 再次调用不会重新初始化
let value = cell.get_or_init(|| "world".to_string());
println!("{}", value);  // "hello"
```

---

## ⚡ 性能考虑

```rust
use std::cell::{Cell, RefCell};

// Cell：零成本，直接内存操作
let cell = Cell::new(0i32);
for _ in 0..1000000 {
    cell.set(cell.get() + 1);  // 编译后和普通变量一样快
}

// RefCell：有运行时检查开销
let cell = RefCell::new(0i32);
for _ in 0..1000000 {
    *cell.borrow_mut() += 1;  // 每次都要检查借用状态
}
```

实际测试中，`RefCell` 的开销很小（通常 1-2 个原子操作），但在极端热路径上可能有影响。

---

## 🎓 课后思考

1. 为什么 `Cell::get()` 要求 `T: Copy`？
2. `RefCell` 的借用检查是如何实现的？（提示：内部有一个计数器）
3. 什么时候应该用 `Rc<RefCell<T>>`，什么时候应该重新设计数据结构？

---

## 🏆 第 100 课里程碑！

恭喜完成 100 课 Rust 学习！回顾一下我们的学习路径：
- 第 1-64 课：语言基础 + 一些第三方库
- 第 65-100 课：回归语言本身，深入标准库

接下来继续探索 std 的更多宝藏！

---

*第 100 课完 🎉*
