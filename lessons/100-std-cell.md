# 🎉 第 100 课：std::cell — 内部可变性

恭喜！我们迎来了第 100 课！今天讲一个 Rust 独有且非常重要的概念：**内部可变性 (Interior Mutability)**。

---

## 🤔 问题：借用规则太严格了？

Rust 的借用规则：
- 要么一个可变引用
- 要么多个不可变引用
- 不能同时存在

但有时候我们需要在**只持有不可变引用**的情况下**修改数据**：

```rust
// ❌ 编译错误！
struct Counter {
    count: i32,
}

impl Counter {
    fn increment(&self) {  // 注意是 &self
        self.count += 1;   // 错误：不能修改 &self
    }
}
```

这在以下场景很常见：
- 缓存/记忆化
- 统计调用次数
- 观察者模式
- 共享状态

---

## 📦 Cell — 简单值的内部可变性

`Cell<T>` 允许在不可变引用下修改值，但只能整体替换：

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<i32>,  // 用 Cell 包装
}

impl Counter {
    fn new() -> Self {
        Counter { count: Cell::new(0) }
    }
    
    fn increment(&self) {  // &self，不是 &mut self
        let old = self.count.get();
        self.count.set(old + 1);
    }
    
    fn get(&self) -> i32 {
        self.count.get()
    }
}

fn main() {
    let counter = Counter::new();  // 不需要 mut
    counter.increment();
    counter.increment();
    println!("Count: {}", counter.get());  // 2
}
```

### Cell 的核心方法

```rust
use std::cell::Cell;

let cell = Cell::new(5);

// get — 获取值（Copy 类型）
let val = cell.get();  // 5

// set — 设置新值
cell.set(10);

// replace — 设置新值，返回旧值
let old = cell.replace(20);  // old = 10

// take — 取出值，留下默认值
let cell = Cell::new(String::from("hello"));
// cell.get();  // ❌ 错误！String 不是 Copy
let val = cell.take();  // val = "hello", cell 变成 ""
```

### ⚠️ Cell 的限制

```rust
// Cell<T> 的 get() 要求 T: Copy
let cell = Cell::new(vec![1, 2, 3]);
// cell.get();  // ❌ Vec 不是 Copy！

// 只能整体替换，不能获取内部引用
let cell = Cell::new(5);
// let r = &cell.???  // ❌ 没有办法获取内部 &i32
```

---

## 📦 RefCell — 运行时借用检查

`RefCell<T>` 把借用检查从编译期推迟到运行时：

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// borrow() — 不可变借用
{
    let r = data.borrow();  // 返回 Ref<Vec<i32>>
    println!("{:?}", *r);   // [1, 2, 3]
}  // Ref 离开作用域，借用结束

// borrow_mut() — 可变借用
{
    let mut r = data.borrow_mut();  // 返回 RefMut<Vec<i32>>
    r.push(4);
}

println!("{:?}", data.borrow());  // [1, 2, 3, 4]
```

### ⚠️ 运行时 panic！

```rust
use std::cell::RefCell;

let data = RefCell::new(5);

let r1 = data.borrow();      // 不可变借用
let r2 = data.borrow_mut();  // 💥 panic! 已经有不可变借用

// 运行时错误：
// thread 'main' panicked at 'already borrowed: BorrowMutError'
```

### 安全的尝试借用

```rust
use std::cell::RefCell;

let data = RefCell::new(5);

let r1 = data.borrow();

// try_borrow_mut 返回 Result，不会 panic
match data.try_borrow_mut() {
    Ok(mut r) => *r += 1,
    Err(_) => println!("借用冲突！"),
}
```

---

## 🆚 Cell vs RefCell

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| 获取引用 | ❌ 只能 get/set | ✅ borrow/borrow_mut |
| 适用类型 | Copy 类型或 take | 任何类型 |
| 检查时机 | 无需检查 | 运行时 |
| panic 风险 | 无 | 有（借用冲突） |
| 开销 | 零 | 小（借用计数） |

**选择原则**：
- 简单的 `Copy` 类型 → `Cell`
- 需要引用内部数据 → `RefCell`

---

## 🧩 实战：缓存计算结果

```rust
use std::cell::RefCell;

struct LazyValue {
    value: RefCell<Option<i32>>,
    computation: fn() -> i32,
}

impl LazyValue {
    fn new(computation: fn() -> i32) -> Self {
        LazyValue {
            value: RefCell::new(None),
            computation,
        }
    }
    
    fn get(&self) -> i32 {  // 只需要 &self
        // 如果已缓存，直接返回
        if let Some(v) = *self.value.borrow() {
            return v;
        }
        
        // 否则计算并缓存
        let v = (self.computation)();
        *self.value.borrow_mut() = Some(v);
        v
    }
}

fn expensive() -> i32 {
    println!("Computing...");
    42
}

fn main() {
    let lazy = LazyValue::new(expensive);
    
    println!("{}", lazy.get());  // 打印 "Computing...", 返回 42
    println!("{}", lazy.get());  // 直接返回 42，不再计算
}
```

---

## 🔄 OnceCell — 只写一次

```rust
use std::cell::OnceCell;

let cell = OnceCell::new();

// 第一次设置成功
assert!(cell.set(1).is_ok());

// 第二次设置失败（值不变）
assert!(cell.set(2).is_err());

println!("{:?}", cell.get());  // Some(1)

// get_or_init — 惰性初始化
let cell: OnceCell<String> = OnceCell::new();
let value = cell.get_or_init(|| {
    println!("初始化中...");
    "hello".to_string()
});
// 第二次调用不会再执行闭包
let value2 = cell.get_or_init(|| unreachable!());
```

---

## 💡 与 PHP/JS 的类比

```php
// PHP：对象默认可变
class Counter {
    private int $count = 0;
    
    public function increment(): void {
        $this->count++;  // 无需 mut，随便改
    }
}
```

```rust
// Rust：必须显式声明可变性
// Cell/RefCell 让你在不可变借用下实现可变

// 这是 Rust 的哲学：
// "可变性是危险的，必须显式声明"
// Cell/RefCell 是安全的逃生通道
```

---

## ⚠️ 重要提醒

1. **Cell 和 RefCell 都是单线程的！** 不能跨线程使用
2. 多线程需要用 `Mutex` / `RwLock`（后面会讲）
3. RefCell 的借用冲突是**运行时 panic**，要小心

```rust
use std::cell::RefCell;

// ❌ 不能跨线程！
let data = RefCell::new(5);
// std::thread::spawn(move || {
//     data.borrow_mut();  // 编译错误！RefCell 不是 Sync
// });
```

---

## 🎓 课后思考

1. 为什么 `Cell<T>` 的 `get()` 要求 `T: Copy`？
2. 什么情况下应该用 `RefCell` 而不是直接用 `&mut`？
3. `OnceCell` 和 `lazy_static!` 宏有什么区别？

---

🎉 **里程碑达成！** 我们已经完成了 100 节 Rust 课程！

感谢大家一路陪伴，后面还有更多精彩内容！

---

*第 100 课完* 🦀
