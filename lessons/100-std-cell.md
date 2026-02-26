# 第 100 课：std::cell — 内部可变性

🎉 里程碑课程！今天讲一个 Rust 独特且重要的概念：**内部可变性 (Interior Mutability)**。

---

## 🤔 问题：借用规则的困境

Rust 的借用规则：
- 要么有多个不可变引用 `&T`
- 要么有一个可变引用 `&mut T`
- 两者不能同时存在

但有时我们需要在只有 `&self` 的情况下修改数据：

```rust
struct Counter {
    value: i32,
}

impl Counter {
    // ❌ 这个方法需要 &mut self
    fn increment(&mut self) {
        self.value += 1;
    }
}

// 问题：如果 Counter 在 Rc 里呢？
use std::rc::Rc;
let counter = Rc::new(Counter { value: 0 });
// counter.increment(); // ❌ Rc 只能给你 &T，不能给 &mut T！
```

这就是内部可变性要解决的问题。

---

## 📦 Cell<T> — 简单值的内部可变性

```rust
use std::cell::Cell;

struct Counter {
    value: Cell<i32>,  // 用 Cell 包装
}

impl Counter {
    fn new() -> Self {
        Counter { value: Cell::new(0) }
    }
    
    // 现在只需要 &self！
    fn increment(&self) {
        let current = self.value.get();
        self.value.set(current + 1);
    }
    
    fn get(&self) -> i32 {
        self.value.get()
    }
}

fn main() {
    let counter = Counter::new();
    counter.increment();  // 用 &self 就能修改
    counter.increment();
    println!("Count: {}", counter.get());  // 2
}
```

### Cell 的特点

```rust
use std::cell::Cell;

let cell = Cell::new(42);

// get() — 获取值的拷贝
let value = cell.get();  // 42

// set() — 设置新值
cell.set(100);

// take() — 取出值，留下默认值
let cell = Cell::new(String::from("hello"));  // ❌ String 没有 Copy！
// cell.get(); // 编译错误！

// Cell<T> 的 get() 只有 T: Copy 时才能用
// 因为它返回的是值的拷贝
```

**Cell 限制**：只适合 `Copy` 类型（数字、bool 等）

---

## 🔓 RefCell<T> — 任意类型的内部可变性

```rust
use std::cell::RefCell;

let cell = RefCell::new(String::from("hello"));

// borrow() — 获取不可变借用 Ref<T>
{
    let s = cell.borrow();
    println!("{}", s);  // "hello"
}  // Ref 离开作用域，借用结束

// borrow_mut() — 获取可变借用 RefMut<T>
{
    let mut s = cell.borrow_mut();
    s.push_str(" world");
}  // RefMut 离开作用域，借用结束

println!("{:?}", cell);  // RefCell { value: "hello world" }
```

### RefCell 的运行时检查

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);

// 借用规则在运行时检查！
let borrow1 = cell.borrow();
let borrow2 = cell.borrow();  // ✅ 多个不可变借用 OK

// let mut_borrow = cell.borrow_mut();  // ❌ panic! 已有不可变借用

drop(borrow1);
drop(borrow2);

let mut_borrow = cell.borrow_mut();  // ✅ 现在可以了
```

**⚠️ RefCell 会在违反借用规则时 panic！**

```rust
let cell = RefCell::new(5);
let r1 = cell.borrow_mut();
let r2 = cell.borrow_mut();  // 💥 panic: already borrowed
```

---

## 🛡️ try_borrow — 安全版本

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);
let r1 = cell.borrow_mut();

// try_borrow_mut 返回 Result，不会 panic
match cell.try_borrow_mut() {
    Ok(mut r) => *r += 1,
    Err(_) => println!("借用冲突！"),
}
```

---

## 🔗 Rc + RefCell = 共享可变数据

这是 Rust 中实现"多个所有者 + 可修改"的经典组合：

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        children: RefCell::new(vec![]),
    });
    
    let branch = Rc::new(Node {
        value: 5,
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });
    
    // 通过 &branch 也能修改 children！
    branch.children.borrow_mut().push(Rc::new(Node {
        value: 7,
        children: RefCell::new(vec![]),
    }));
    
    println!("{:?}", branch);
}
```

### 实际例子：共享缓存

```rust
use std::rc::Rc;
use std::cell::RefCell;
use std::collections::HashMap;

struct Cache {
    data: RefCell<HashMap<String, String>>,
}

impl Cache {
    fn new() -> Rc<Self> {
        Rc::new(Cache {
            data: RefCell::new(HashMap::new()),
        })
    }
    
    fn get(&self, key: &str) -> Option<String> {
        self.data.borrow().get(key).cloned()
    }
    
    fn set(&self, key: String, value: String) {
        self.data.borrow_mut().insert(key, value);
    }
}

fn main() {
    let cache = Cache::new();
    let cache2 = Rc::clone(&cache);
    
    cache.set("name".into(), "Rust".into());
    println!("{:?}", cache2.get("name"));  // Some("Rust")
}
```

---

## 📊 Cell vs RefCell

| 特性 | Cell<T> | RefCell<T> |
|------|----------|-------------|
| T 的要求 | 通常 T: Copy | 任意类型 |
| 访问方式 | get/set（拷贝） | borrow/borrow_mut（引用） |
| 开销 | 零开销 | 运行时借用检查 |
| 失败处理 | 编译时 | 运行时 panic |

**选择原则**：
- 简单 Copy 类型 → `Cell`
- 复杂类型 → `RefCell`

---

## ⚡ OnceCell — 一次性初始化

```rust
use std::cell::OnceCell;

let cell: OnceCell<String> = OnceCell::new();

// 只能设置一次
assert!(cell.set("hello".into()).is_ok());
assert!(cell.set("world".into()).is_err());  // 第二次失败

// get 返回 Option<&T>
println!("{:?}", cell.get());  // Some("hello")

// get_or_init — 懒初始化
let cell: OnceCell<i32> = OnceCell::new();
let value = cell.get_or_init(|| {
    println!("计算中...");
    42
});
println!("{}", value);  // 42

// 第二次调用不会重新计算
let value = cell.get_or_init(|| 100);
println!("{}", value);  // 还是 42
```

---

## 🧵 多线程版本

单线程用 `Cell`/`RefCell`，多线程用：
- `Cell` → `AtomicXxx` (如 `AtomicI32`)
- `RefCell` → `Mutex` 或 `RwLock`
- `OnceCell` → `OnceLock`

```rust
// 单线程
use std::cell::RefCell;
let data = RefCell::new(vec![1, 2, 3]);

// 多线程
use std::sync::Mutex;
let data = Mutex::new(vec![1, 2, 3]);
```

---

## 🎓 课后思考

1. 为什么 `Cell` 和 `RefCell` 不能跨线程使用？
2. 在什么场景下 `RefCell` 的运行时检查会 panic？如何避免？
3. `Rc<RefCell<T>>` 这个组合为什么这么常见？

---

*🎉 恭喜完成第 100 课！我们已经系统学习了 Rust 语言和标准库的核心内容。继续加油！*
