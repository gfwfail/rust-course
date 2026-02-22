# 第 78 课：UnsafeCell — 内部可变性的基石

> 日期：2026-02-22  
> 主题：std::cell::UnsafeCell 的原理与使用

---

## 回顾：Rust 的借用规则

Rust 有一条铁律：

```rust
// ❌ 不能同时存在可变引用和不可变引用
let mut x = 42;
let r1 = &x;      // 不可变借用
let r2 = &mut x;  // 💥 编译错误！
```

但我们之前学过 `Cell<T>` 和 `RefCell<T>`，它们能"绕过"这个规则：

```rust
use std::cell::Cell;

let x = Cell::new(42);  // 注意：x 不是 mut
x.set(100);  // 但可以修改！🤔
```

**这是怎么做到的？** 答案就是今天的主角：`UnsafeCell<T>`。

---

## UnsafeCell 是什么？

```rust
use std::cell::UnsafeCell;

// UnsafeCell 的定义非常简单：
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}
```

看起来就是包了一层，但它有一个**魔法属性**：

**UnsafeCell 是 Rust 中唯一合法的方式，通过 `&T` 获得 `*mut T`**

```rust
use std::cell::UnsafeCell;

let cell = UnsafeCell::new(42);

// 即使只有 &cell（不可变引用）
// 也能获取内部的可变指针！
let ptr: *mut i32 = cell.get();

unsafe {
    *ptr = 100;  // 修改成功
}
```

---

## 为什么需要 UnsafeCell？

Rust 编译器做了一个重要假设：

> 如果你只有 `&T`（不可变引用），那么 T 的内容**不会**被修改

编译器基于这个假设做优化，比如：

```rust
fn read_twice(x: &i32) -> i32 {
    let a = *x;
    // 编译器可能优化：既然 x 是不可变引用，
    // 第二次读取肯定和第一次一样，直接复用 a
    let b = *x;
    a + b
}
```

但 `UnsafeCell` 告诉编译器：**这个类型的内容可能会变，不要做那些假设！**

```rust
use std::cell::UnsafeCell;

fn read_twice(x: &UnsafeCell<i32>) -> i32 {
    unsafe {
        let a = *x.get();
        // 编译器不会优化，因为知道值可能变了
        let b = *x.get();
        a + b
    }
}
```

---

## UnsafeCell 的核心方法

```rust
use std::cell::UnsafeCell;

let mut cell = UnsafeCell::new(42);

// get() - 获取可变裸指针（即使只有 &self）
let ptr: *mut i32 = cell.get();

// get_mut() - 获取可变引用（需要 &mut self）
let r: &mut i32 = cell.get_mut();

// into_inner() - 消耗 cell，取出值
let value: i32 = cell.into_inner();
```

**关键区别**：
- `get()` 只需要 `&self`，返回 `*mut T`（这就是魔法所在！）
- `get_mut()` 需要 `&mut self`，返回 `&mut T`（普通借用规则）

---

## Cell 是如何基于 UnsafeCell 实现的

```rust
use std::cell::UnsafeCell;

// 简化版 Cell 实现
pub struct MyCell<T> {
    value: UnsafeCell<T>,
}

impl<T: Copy> MyCell<T> {
    pub fn new(value: T) -> Self {
        MyCell {
            value: UnsafeCell::new(value),
        }
    }
    
    pub fn get(&self) -> T {
        // 通过 UnsafeCell 获取指针，然后读取
        unsafe { *self.value.get() }
    }
    
    pub fn set(&self, value: T) {
        // 通过 UnsafeCell 获取指针，然后写入
        unsafe { *self.value.get() = value; }
    }
}

// 使用
let cell = MyCell::new(42);
cell.set(100);  // &self 就能修改！
println!("{}", cell.get());  // 100
```

**注意**：`Cell<T>` 要求 `T: Copy`，这保证了读写操作是原子的（按位复制）。

---

## RefCell 是如何基于 UnsafeCell 实现的

```rust
use std::cell::{UnsafeCell, Cell};

// 简化版 RefCell 实现
pub struct MyRefCell<T> {
    value: UnsafeCell<T>,
    borrow_state: Cell<isize>,  // 借用计数
    // > 0: 有 N 个不可变借用
    // = 0: 无借用
    // = -1: 有一个可变借用
}

impl<T> MyRefCell<T> {
    pub fn borrow(&self) -> Ref<T> {
        // 检查：不能在可变借用期间借用
        let state = self.borrow_state.get();
        if state < 0 {
            panic!("already mutably borrowed");
        }
        self.borrow_state.set(state + 1);
        
        Ref {
            value: unsafe { &*self.value.get() },
            borrow_state: &self.borrow_state,
        }
    }
    
    pub fn borrow_mut(&self) -> RefMut<T> {
        // 检查：不能有任何借用
        if self.borrow_state.get() != 0 {
            panic!("already borrowed");
        }
        self.borrow_state.set(-1);
        
        RefMut {
            value: unsafe { &mut *self.value.get() },
            borrow_state: &self.borrow_state,
        }
    }
}
```

**RefCell 把编译时检查换成了运行时检查**，但底层仍是 UnsafeCell。

---

## 实际应用：自定义自旋锁

```rust
use std::cell::UnsafeCell;
use std::sync::atomic::{AtomicBool, Ordering};

// 一个简单的自旋锁
pub struct SpinLock<T> {
    locked: AtomicBool,
    data: UnsafeCell<T>,
}

// 告诉编译器这个类型可以跨线程共享
unsafe impl<T: Send> Sync for SpinLock<T> {}

impl<T> SpinLock<T> {
    pub fn new(data: T) -> Self {
        SpinLock {
            locked: AtomicBool::new(false),
            data: UnsafeCell::new(data),
        }
    }
    
    pub fn lock(&self) -> SpinLockGuard<T> {
        // 自旋等待获取锁
        while self.locked
            .compare_exchange_weak(
                false, true,
                Ordering::Acquire,
                Ordering::Relaxed
            )
            .is_err()
        {
            // 自旋
            std::hint::spin_loop();
        }
        
        SpinLockGuard { lock: self }
    }
}

pub struct SpinLockGuard<'a, T> {
    lock: &'a SpinLock<T>,
}

impl<T> std::ops::Deref for SpinLockGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        unsafe { &*self.lock.data.get() }
    }
}

impl<T> std::ops::DerefMut for SpinLockGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.lock.data.get() }
    }
}

impl<T> Drop for SpinLockGuard<'_, T> {
    fn drop(&mut self) {
        self.lock.locked.store(false, Ordering::Release);
    }
}
```

---

## UnsafeCell 的安全规则

虽然叫 `Unsafe`Cell，但使用时必须遵守规则：

```rust
use std::cell::UnsafeCell;

let cell = UnsafeCell::new(42);

// ❌ 错误：不能同时有多个 &mut
unsafe {
    let r1 = &mut *cell.get();
    let r2 = &mut *cell.get();  // UB!
    *r1 = 1;
    *r2 = 2;
}

// ❌ 错误：不能在有 &T 时修改
unsafe {
    let r = &*cell.get();  // 创建 &T
    *cell.get() = 100;     // 同时修改，UB!
    println!("{}", r);
}

// ✅ 正确：一次只有一个引用
unsafe {
    *cell.get() = 100;  // 修改
    let r = &*cell.get();  // 然后读取
    println!("{}", r);
}
```

**关键**：UnsafeCell 绕过了编译器检查，但你仍要遵守借用规则。违反规则 = UB。

---

## 与 PHP/JS 对比

```php
// PHP - 一切都是"内部可变"的
class Counter {
    public $count = 0;
}

$c = new Counter();
$ref = $c;  // 引用
$c->count = 10;  // 随便改
echo $ref->count;  // 10

// 没有借用检查，没有线程安全保证
```

```rust
// Rust - 默认不可变，内部可变需要显式声明
use std::cell::RefCell;

struct Counter {
    count: RefCell<i32>,  // 明确声明"这个字段可以内部可变"
}

let c = Counter { count: RefCell::new(0) };
*c.count.borrow_mut() = 10;
println!("{}", c.count.borrow());  // 10
```

**Rust 的优势**：
- 内部可变是 **显式的**，看到 Cell/RefCell 就知道"这里会变"
- 有借用检查（RefCell 运行时检查，Cell 只允许 Copy 类型）
- 线程安全有保证（Cell/RefCell 不是 Sync）

---

## UnsafeCell 的类型属性

```rust
use std::cell::UnsafeCell;

// UnsafeCell<T> 不是 Sync
// 即使 T 是 Sync
fn test_sync<T: Sync>() {}

// test_sync::<UnsafeCell<i32>>();  // ❌ 编译错误！

// 这是故意的：UnsafeCell 的内部可变性
// 在多线程下不安全（没有同步机制）

// 如果要多线程共享，用 Mutex 或 RwLock
// 它们内部使用 UnsafeCell + 同步原语
```

---

## 总结

| 概念 | 说明 |
|------|------|
| `UnsafeCell<T>` | 唯一合法的 `&T` → `*mut T` 途径 |
| `.get()` | 从 `&self` 获取 `*mut T` |
| 编译器 | 看到 UnsafeCell 会禁用某些优化 |
| `Cell<T>` | UnsafeCell + Copy 语义 |
| `RefCell<T>` | UnsafeCell + 运行时借用检查 |
| `Mutex<T>` | UnsafeCell + 互斥锁 |
| 安全规则 | 仍要遵守借用规则，否则 UB |

**使用场景**：
- 实现自定义同步原语（锁、原子容器）
- 实现内部可变性容器
- 底层数据结构
- FFI 交互

**记住**：
- 内部可变性的所有类型（Cell, RefCell, Mutex, RwLock...）都基于 UnsafeCell
- UnsafeCell 是"逃生舱口"，让你告诉编译器"我知道我在做什么"
- 使用 UnsafeCell 时，借用规则的检查责任从编译器转移到你身上

---

*下节课：OnceCell 与 LazyCell — 延迟初始化*
