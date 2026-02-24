# 🎉 第 100 课：std::cell — 内部可变性 (Interior Mutability)

恭喜大家！我们迎来了第 100 课的里程碑！今天讲一个核心概念：**内部可变性**。

---

## 🤔 问题：为什么需要内部可变性？

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

**问题来了**：如果我有一个 `&T`（不可变引用），但想修改里面的某些字段怎么办？

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn increment(&self) {  // 注意：&self，不是 &mut self
        self.count += 1;   // ❌ 编译错误！不能通过 &self 修改
    }
}
```

这时候就需要 **内部可变性**！

---

## 📦 Cell — 单线程简单值

`Cell<T>` 适用于 `Copy` 类型的简单值：

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<u32>,  // 包在 Cell 里
}

impl Counter {
    fn new() -> Self {
        Counter { count: Cell::new(0) }
    }
    
    fn increment(&self) {  // 仍然是 &self
        let old = self.count.get();  // 读取
        self.count.set(old + 1);     // 设置
    }
    
    fn get(&self) -> u32 {
        self.count.get()
    }
}

fn main() {
    let counter = Counter::new();
    counter.increment();
    counter.increment();
    println!("Count: {}", counter.get());  // 2
}
```

### Cell 的特点

```rust
use std::cell::Cell;

let cell = Cell::new(42);

// get() 返回值的拷贝（要求 T: Copy）
let value = cell.get();

// set() 替换值
cell.set(100);

// take() 取出值，留下默认值（要求 T: Default）
let cell = Cell::new(Some(42));
let taken = cell.take();  // Some(42)
assert_eq!(cell.get(), None);

// replace() 替换并返回旧值
let old = cell.replace(Some(99));
```

**⚠️ 限制**：`Cell` 只能整体替换值，不能借用内部引用！

---

## 🔄 RefCell — 运行时借用检查

`RefCell<T>` 更灵活，可以借用内部引用：

```rust
use std::cell::RefCell;

let cell = RefCell::new(vec![1, 2, 3]);

// borrow() 获取不可变引用
{
    let vec = cell.borrow();
    println!("First: {}", vec[0]);
}  // vec 离开作用域，借用结束

// borrow_mut() 获取可变引用
{
    let mut vec = cell.borrow_mut();
    vec.push(4);
}

println!("{:?}", cell.borrow());  // [1, 2, 3, 4]
```

### ⚠️ 运行时借用检查

`RefCell` 把借用检查从编译时移到运行时：

```rust
use std::cell::RefCell;

let cell = RefCell::new(42);

let r1 = cell.borrow();
let r2 = cell.borrow();  // ✅ 多个不可变借用

// let r3 = cell.borrow_mut();  // ❌ panic! 已有不可变借用

drop(r1);
drop(r2);

let mut r3 = cell.borrow_mut();  // ✅ 现在可以了
// let r4 = cell.borrow();  // ❌ panic! 已有可变借用
```

### try_borrow 系列 — 不 panic

```rust
use std::cell::RefCell;

let cell = RefCell::new(42);
let r1 = cell.borrow();

// try_borrow_mut 返回 Result
match cell.try_borrow_mut() {
    Ok(mut r) => *r = 100,
    Err(_) => println!("借用冲突，跳过修改"),
}
```

---

## 🆚 Cell vs RefCell

| 特性 | Cell | RefCell |
|-----|------|---------|
| 适用类型 | T: Copy | 任意 T |
| 获取引用 | ❌ 只能 get/set | ✅ borrow/borrow_mut |
| 检查时机 | 无需检查 | 运行时检查 |
| panic 风险 | 无 | 借用冲突会 panic |
| 性能 | 更快（零开销） | 有运行时开销 |

**选择原则**：
- 简单 Copy 类型（i32, bool, Option<usize>）→ `Cell`
- 复杂类型需要借用 → `RefCell`

---

## 💡 实际应用场景

### 1️⃣ 缓存计算结果（Memoization）

```rust
use std::cell::RefCell;

struct Fibonacci {
    cache: RefCell<Vec<Option<u64>>>,
}

impl Fibonacci {
    fn new(size: usize) -> Self {
        Fibonacci {
            cache: RefCell::new(vec![None; size]),
        }
    }
    
    fn get(&self, n: usize) -> u64 {  // &self，但能修改缓存
        // 先检查缓存
        if let Some(value) = self.cache.borrow().get(n).copied().flatten() {
            return value;
        }
        
        // 计算
        let result = if n <= 1 {
            n as u64
        } else {
            self.get(n - 1) + self.get(n - 2)
        };
        
        // 存入缓存
        if n < self.cache.borrow().len() {
            self.cache.borrow_mut()[n] = Some(result);
        }
        
        result
    }
}

fn main() {
    let fib = Fibonacci::new(100);
    println!("fib(50) = {}", fib.get(50));
}
```

### 2️⃣ 共享可变状态（配合 Rc）

```rust
use std::cell::RefCell;
use std::rc::Rc;

// 多个所有者共享可变数据
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

let a = Rc::clone(&shared);
let b = Rc::clone(&shared);

a.borrow_mut().push(4);
b.borrow_mut().push(5);

println!("{:?}", shared.borrow());  // [1, 2, 3, 4, 5]
```

### 3️⃣ 观察者模式

```rust
use std::cell::RefCell;
use std::rc::Rc;

type Callback = Box<dyn Fn(i32)>;

struct Observable {
    value: Cell<i32>,
    listeners: RefCell<Vec<Callback>>,
}

impl Observable {
    fn new(value: i32) -> Self {
        Observable {
            value: Cell::new(value),
            listeners: RefCell::new(Vec::new()),
        }
    }
    
    fn subscribe(&self, callback: Callback) {
        self.listeners.borrow_mut().push(callback);
    }
    
    fn set(&self, value: i32) {
        self.value.set(value);
        for listener in self.listeners.borrow().iter() {
            listener(value);
        }
    }
}
```

---

## 🔐 OnceCell — 一次性初始化

```rust
use std::cell::OnceCell;

let cell: OnceCell<String> = OnceCell::new();

// 第一次设置成功
assert!(cell.set("hello".to_string()).is_ok());

// 第二次设置失败（已有值）
assert!(cell.set("world".to_string()).is_err());

// 获取值
println!("{}", cell.get().unwrap());  // "hello"

// get_or_init — 懒初始化
let cell: OnceCell<String> = OnceCell::new();
let value = cell.get_or_init(|| {
    println!("初始化中...");
    "computed".to_string()
});
// 第二次调用不会再初始化
let value2 = cell.get_or_init(|| "another".to_string());
assert_eq!(value, value2);
```

**应用：延迟初始化全局配置**

```rust
use std::cell::OnceCell;

struct Config {
    db_url: String,
}

thread_local! {
    static CONFIG: OnceCell<Config> = OnceCell::new();
}

fn get_config() -> &'static Config {
    CONFIG.with(|cell| {
        cell.get_or_init(|| Config {
            db_url: std::env::var("DATABASE_URL")
                .unwrap_or_else(|_| "localhost".to_string())
        })
    })
}
```

---

## ⚠️ 注意事项

### 1. Cell/RefCell 不是线程安全的！

```rust
use std::cell::RefCell;

let cell = RefCell::new(42);

// ❌ 编译错误：RefCell 不能跨线程
// std::thread::spawn(move || {
//     cell.borrow_mut();
// });

// 多线程用 Mutex / RwLock（下节课讲）
```

### 2. RefCell 借用冲突会 panic

```rust
let cell = RefCell::new(42);
let r1 = cell.borrow_mut();
let r2 = cell.borrow();  // 💥 panic at runtime!
```

### 3. 避免在借用期间调用可能再次借用的代码

```rust
let cell = RefCell::new(vec![1, 2, 3]);

// ❌ 危险：在借用期间调用可能再次借用的闭包
let r = cell.borrow();
// some_function_that_might_borrow(&cell);  // 可能 panic
```

---

## 🧠 总结

| 类型 | 用途 | 线程安全 |
|------|------|----------|
| `Cell<T>` | Copy 类型的内部可变性 | ❌ |
| `RefCell<T>` | 运行时借用检查 | ❌ |
| `OnceCell<T>` | 一次性初始化 | ❌ |
| `Mutex<T>` | 多线程互斥锁 | ✅ |
| `RwLock<T>` | 多线程读写锁 | ✅ |

**内部可变性的本质**：把 Rust 的借用检查从编译时推迟到运行时，由程序员保证不会出现借用冲突。

---

## 🎓 课后思考

1. 什么时候应该用 `Cell` 而不是 `RefCell`？
2. `Rc<RefCell<T>>` 这个组合为什么这么常见？
3. 如果需要线程安全的内部可变性，应该用什么？

---

🎉 **第 100 课完！我们已经走过了 100 节课的 Rust 旅程！**

下节课我们讲 `std::mem` — 内存操作黑魔法。
