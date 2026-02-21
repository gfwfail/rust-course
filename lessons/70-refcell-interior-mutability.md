# 第 70 课：RefCell 与内部可变性

> 日期：2026-02-21  
> 主题：RefCell<T>, 内部可变性, borrow(), borrow_mut()

---

## 回顾上节课的问题

`Rc<T>` 和 `Arc<T>` 只给你不可变引用 `&T`。

```rust
use std::rc::Rc;

let data = Rc::new(vec![1, 2, 3]);

// ❌ 编译错误！
data.push(4);  // 不能修改！
```

但如果我**就是需要**多个所有者 + 可以修改呢？

答案：**RefCell<T> — 内部可变性**

---

## 什么是内部可变性？

Rust 的借用规则（编译时检查）：
- 要么 1 个 `&mut T`（独占写）
- 要么 N 个 `&T`（共享读）

**内部可变性**：把这个检查推迟到**运行时**！

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// 外面看起来是不可变的，但内部可以修改！
data.borrow_mut().push(4);

println!("{:?}", data.borrow()); // [1, 2, 3, 4]
```

---

## RefCell 的 API

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);

// 不可变借用 → 返回 Ref<T>
let r = cell.borrow();
println!("值: {}", *r);
drop(r);  // 记得释放！

// 可变借用 → 返回 RefMut<T>
let mut m = cell.borrow_mut();
*m = 10;
drop(m);

println!("新值: {}", cell.borrow()); // 10
```

---

## 运行时借用检查

RefCell 在**运行时**检查借用规则：

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);

let r1 = cell.borrow();     // ✅ 第一个不可变借用
let r2 = cell.borrow();     // ✅ 第二个不可变借用（OK）

// ❌ 运行时 panic！已经有不可变借用了
// let m = cell.borrow_mut();

drop(r1);
drop(r2);

let m = cell.borrow_mut();  // ✅ 现在可以了
```

**违反规则 = panic，不是编译错误！**

---

## try_borrow 避免 panic

```rust
use std::cell::RefCell;

let cell = RefCell::new(5);
let r = cell.borrow();

// 用 try_borrow_mut 检查而不是 panic
match cell.try_borrow_mut() {
    Ok(mut m) => *m = 10,
    Err(_) => println!("已经被借用了！"),
}
```

---

## Rc + RefCell = 黄金搭档

共享所有权 + 可修改：

```rust
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    // 多个所有者，且可以修改
    let data = Rc::new(RefCell::new(vec![1, 2, 3]));
    
    let data2 = Rc::clone(&data);
    let data3 = Rc::clone(&data);
    
    // 任何一个都可以修改！
    data.borrow_mut().push(4);
    data2.borrow_mut().push(5);
    
    println!("{:?}", data3.borrow()); // [1, 2, 3, 4, 5]
}
```

**这个模式超级常用！**

---

## 实战：观察者模式

```rust
use std::rc::Rc;
use std::cell::RefCell;

trait Observer {
    fn update(&self, value: i32);
}

struct Subject {
    observers: Vec<Rc<dyn Observer>>,
    state: i32,
}

impl Subject {
    fn new() -> Self {
        Subject { observers: vec![], state: 0 }
    }
    
    fn add_observer(&mut self, o: Rc<dyn Observer>) {
        self.observers.push(o);
    }
    
    fn set_state(&mut self, value: i32) {
        self.state = value;
        for o in &self.observers {
            o.update(value);
        }
    }
}

// 观察者需要修改自己的状态
struct Logger {
    logs: RefCell<Vec<String>>,  // 用 RefCell 实现内部可变
}

impl Observer for Logger {
    fn update(&self, value: i32) {
        // &self 是不可变引用，但我们能修改 logs！
        self.logs.borrow_mut().push(format!("收到: {}", value));
    }
}

fn main() {
    let logger = Rc::new(Logger {
        logs: RefCell::new(vec![]),
    });
    
    let mut subject = Subject::new();
    subject.add_observer(Rc::clone(&logger) as Rc<dyn Observer>);
    
    subject.set_state(42);
    subject.set_state(100);
    
    println!("日志: {:?}", logger.logs.borrow());
    // ["收到: 42", "收到: 100"]
}
```

---

## Cell<T> — 更简单的版本

`Cell<T>` 用于 Copy 类型，更轻量：

```rust
use std::cell::Cell;

let cell = Cell::new(5);

// 直接 get/set，不需要借用
println!("值: {}", cell.get());  // 5
cell.set(10);
println!("值: {}", cell.get());  // 10
```

**Cell vs RefCell：**

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| T 的要求 | 必须是 Copy | 任意类型 |
| 访问方式 | get() 返回复制 | borrow() 返回引用 |
| 借用检查 | 无 | 运行时 |
| 性能 | 更轻量 | 更灵活 |

```rust
// ✅ Cell 适合简单值
let counter = Cell::new(0);
counter.set(counter.get() + 1);

// ✅ RefCell 适合复杂类型
let list = RefCell::new(Vec::new());
list.borrow_mut().push(42);
```

---

## 多线程怎么办？

⚠️ `RefCell` 不是线程安全的！

多线程用 `Mutex<T>` 或 `RwLock<T>`（后面会讲）：

| 单线程 | 多线程 |
|--------|--------|
| RefCell<T> | Mutex<T> |
| Cell<T> | AtomicXxx |

```rust
// ❌ RefCell 不能跨线程
// ✅ 用 Arc<Mutex<T>> 代替
use std::sync::{Arc, Mutex};

let data = Arc::new(Mutex::new(vec![1, 2, 3]));
```

---

## 对比 PHP/Laravel

PHP 里随便改对象属性：
```php
class User {
    public $name = "Alice";
}
$user = new User();
$user->name = "Bob";  // 随便改
```

Rust 需要明确可变性：
```rust
// 方案 1：直接 mut
let mut user = User { name: String::from("Alice") };
user.name = String::from("Bob");

// 方案 2：内部可变性
let user = User { name: RefCell::new(String::from("Alice")) };
*user.name.borrow_mut() = String::from("Bob");
```

**为什么这么麻烦？**
- 编译器能帮你发现数据竞争
- 代码意图更清晰
- 并发 bug 更少

---

## 何时用 RefCell？

✅ **适合场景：**
- 实现观察者/回调模式（&self 方法需要修改状态）
- Mock 测试（追踪调用次数）
- 图/树结构（节点互相引用）
- 你**确定**借用规则正确，但编译器不信你

❌ **避免场景：**
- 能用普通 `&mut` 解决的
- 多线程（用 Mutex）
- 滥用来绕过借用检查（设计可能有问题）

---

## 要点总结

| 类型 | 用途 | 检查时机 |
|------|------|----------|
| 普通借用 | 默认方式 | 编译时 |
| `Cell<T>` | Copy 类型内部可变 | 无 |
| `RefCell<T>` | 任意类型内部可变 | 运行时 |
| `Mutex<T>` | 多线程内部可变 | 运行时 |

**记住：**
- `RefCell` = 运行时借用检查
- `Rc<RefCell<T>>` = 共享所有权 + 可修改
- 单线程用 RefCell，多线程用 Mutex

---

## 下节预告

**Option 与 Result 深入**

标准库最重要的两个枚举，它们的方法你用对了吗？

---

*课程笔记：性奴001*
