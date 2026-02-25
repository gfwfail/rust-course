# 第 100 课：std::cell — 内部可变性

🎉 恭喜！我们到达了第 100 课的里程碑！今天讲一个 Rust 所有权系统的重要补充：**内部可变性 (Interior Mutability)**。

---

## 🤔 问题：所有权规则的限制

Rust 的借用规则很严格：
- 要么有多个不可变引用 `&T`
- 要么有一个可变引用 `&mut T`
- 不能同时存在

但有时候我们需要"看起来不可变，实际上可以修改"：

```rust
// ❌ 编译错误
struct Counter {
    count: i32,
}

impl Counter {
    // &self 是不可变引用，不能修改 count
    fn increment(&self) {
        self.count += 1;  // 错误！
    }
}
```

这时候就需要 **内部可变性**。

---

## 📦 Cell<T> — 简单值的内部可变性

`Cell<T>` 适用于 `Copy` 类型（如 `i32`、`bool`）：

```rust
use std::cell::Cell;

struct Counter {
    count: Cell<i32>,
}

impl Counter {
    fn new() -> Self {
        Counter { count: Cell::new(0) }
    }
    
    // 现在 &self 也能修改了！
    fn increment(&self) {
        let current = self.count.get();
        self.count.set(current + 1);
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

### Cell 的 API

```rust
use std::cell::Cell;

let cell = Cell::new(5);

// 获取值（复制出来）
let value = cell.get();  // 5

// 设置值
cell.set(10);

// 替换并返回旧值
let old = cell.replace(20);  // old = 10

// 取出值（消耗 Cell）
let inner = cell.into_inner();  // 20
```

**限制：** `Cell` 只能用于 `Copy` 类型，因为 `get()` 需要复制值。

---

## 🔓 RefCell<T> — 运行时借用检查

`RefCell<T>` 适用于任何类型，但借用检查推迟到运行时：

```rust
use std::cell::RefCell;

struct Document {
    content: RefCell<String>,
    modified: Cell<bool>,
}

impl Document {
    fn new(text: &str) -> Self {
        Document {
            content: RefCell::new(text.to_string()),
            modified: Cell::new(false),
        }
    }
    
    // &self 却能修改内容
    fn append(&self, text: &str) {
        // borrow_mut() 返回可变引用
        self.content.borrow_mut().push_str(text);
        self.modified.set(true);
    }
    
    fn read(&self) -> String {
        // borrow() 返回不可变引用
        self.content.borrow().clone()
    }
}

fn main() {
    let doc = Document::new("Hello");
    doc.append(", World!");
    println!("{}", doc.read());  // "Hello, World!"
}
```

### RefCell 的 API

```rust
use std::cell::RefCell;

let cell = RefCell::new(vec![1, 2, 3]);

// 不可变借用
let borrowed = cell.borrow();
println!("{:?}", *borrowed);  // [1, 2, 3]
drop(borrowed);  // 显式释放借用

// 可变借用
let mut borrowed_mut = cell.borrow_mut();
borrowed_mut.push(4);
drop(borrowed_mut);

// try_borrow / try_borrow_mut 返回 Result
if let Ok(r) = cell.try_borrow() {
    println!("借用成功");
}
```

---

## ⚠️ RefCell 的运行时 Panic

编译器不检查 RefCell 的借用规则，违反规则会 **运行时 panic**：

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);

let r1 = cell.borrow();      // 不可变借用
let r2 = cell.borrow();      // ✅ 多个不可变借用 OK

// let r3 = cell.borrow_mut();  
// ❌ panic! 已有不可变借用时不能可变借用

drop(r1);
drop(r2);

let r4 = cell.borrow_mut();  // ✅ 现在可以了
// let r5 = cell.borrow_mut();  
// ❌ panic! 同时两个可变借用
```

**规则和编译期一样**，只是检查时机变成运行时。

---

## 🎯 Cell vs RefCell 对比

| 特性 | `Cell<T>` | `RefCell<T>` |
|------|-----------|--------------|
| 类型限制 | 只能 `Copy` 类型 | 任何类型 |
| 获取方式 | `get()` 复制值 | `borrow()` 返回引用 |
| 修改方式 | `set()` 设置新值 | `borrow_mut()` 可变引用 |
| 性能 | 零开销 | 运行时借用计数 |
| 线程安全 | ❌ | ❌ |

**选择原则：**
- 简单 Copy 类型（i32、bool） → `Cell`
- 复杂类型（String、Vec） → `RefCell`

---

## 🧩 实战：带缓存的计算

```rust
use std::cell::RefCell;

struct Fibonacci {
    cache: RefCell<Vec<u64>>,
}

impl Fibonacci {
    fn new() -> Self {
        Fibonacci {
            cache: RefCell::new(vec![0, 1]),
        }
    }
    
    // &self 方法，但内部修改缓存
    fn get(&self, n: usize) -> u64 {
        // 先检查缓存
        if let Some(&val) = self.cache.borrow().get(n) {
            return val;
        }
        
        // 需要计算
        let result = self.get(n - 1) + self.get(n - 2);
        
        // 更新缓存
        self.cache.borrow_mut().push(result);
        result
    }
}

fn main() {
    let fib = Fibonacci::new();
    println!("fib(10) = {}", fib.get(10));  // 55
    println!("fib(20) = {}", fib.get(20));  // 6765
}
```

---

## 🔗 Rc + RefCell 组合

单线程下共享可变数据的经典模式：

```rust
use std::rc::Rc;
use std::cell::RefCell;

type SharedVec = Rc<RefCell<Vec<i32>>>;

fn main() {
    let data: SharedVec = Rc::new(RefCell::new(vec![1, 2, 3]));
    
    // 克隆引用（共享所有权）
    let data2 = Rc::clone(&data);
    
    // 通过 data2 修改
    data2.borrow_mut().push(4);
    
    // 通过 data 读取
    println!("{:?}", data.borrow());  // [1, 2, 3, 4]
}
```

类比 PHP/JS：这就像多个变量指向同一个对象，都能修改它。

---

## 🧠 为什么需要内部可变性？

1. **缓存/Memoization** — 计算结果缓存
2. **引用计数** — `Rc` 内部需要修改计数
3. **懒初始化** — `OnceCell`（稍后讲）
4. **Mock/测试** — 记录调用次数
5. **循环引用** — `Rc<RefCell<T>>` 构建图结构

---

## 🎓 课后思考

1. 为什么 `Cell` 和 `RefCell` 都不是线程安全的？
2. `RefCell` 的运行时检查有什么开销？
3. 如果需要线程安全的内部可变性，应该用什么？（提示：`Mutex`）

---

*🎊 恭喜完成第 100 课！下一课我们继续探索标准库的其他宝藏。*
