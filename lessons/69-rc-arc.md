# 第 69 课：Rc 与 Arc — 共享所有权

> 日期：2026-02-21  
> 主题：Rc<T>, Arc<T>, Weak<T>, 引用计数

---

## 今天的问题

Rust 的所有权规则：**一个值只能有一个所有者**。

但如果我需要多个变量共享同一个数据呢？

比如：图的节点、树的节点被多个父节点引用...

答案：**引用计数智能指针 `Rc<T>` 和 `Arc<T>`**

---

## Rc<T> — 单线程引用计数

`Rc` = Reference Counted（引用计数）

```rust
use std::rc::Rc;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);
    
    // 创建第二个所有者
    let data2 = Rc::clone(&data);
    
    // 两个变量指向同一个 Vec
    println!("{:?}", data);   // [1, 2, 3]
    println!("{:?}", data2);  // [1, 2, 3]
    
    // 查看引用计数
    println!("计数: {}", Rc::strong_count(&data)); // 2
}
```

**关键点：`Rc::clone()` 不复制数据，只增加引用计数！**

---

## 引用计数的工作原理

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("hello"));
    println!("创建 a, 计数 = {}", Rc::strong_count(&a)); // 1
    
    {
        let b = Rc::clone(&a);
        println!("创建 b, 计数 = {}", Rc::strong_count(&a)); // 2
        
        let c = Rc::clone(&a);
        println!("创建 c, 计数 = {}", Rc::strong_count(&a)); // 3
    } // b, c 离开作用域
    
    println!("b,c 离开, 计数 = {}", Rc::strong_count(&a)); // 1
} // a 离开，计数变 0，数据被释放
```

**规则：计数归零时，数据自动释放**

---

## Rc::clone vs .clone()

```rust
// ✅ 推荐写法：清晰表达意图
let data2 = Rc::clone(&data);

// ⚠️ 也能工作，但容易误解
let data2 = data.clone();
```

用 `Rc::clone()` 明确告诉读者："我只是增加引用计数，不是深拷贝数据！"

---

## Rc 是不可变的！

```rust
use std::rc::Rc;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);
    
    // ❌ 编译错误！Rc 内的数据不能修改
    // data.push(4);
    
    // Rc<T> 只给你 &T，不给 &mut T
}
```

需要可变性？配合 `RefCell` 使用（下节课讲）。

---

## Arc<T> — 多线程版本

`Arc` = Atomically Reference Counted（原子引用计数）

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);
    
    let data_clone = Arc::clone(&data);
    
    let handle = thread::spawn(move || {
        println!("线程里: {:?}", data_clone);
    });
    
    println!("主线程: {:?}", data);
    handle.join().unwrap();
}
```

**Rc 不能跨线程，Arc 可以！**

---

## Rc vs Arc

| 特性 | Rc<T> | Arc<T> |
|------|-------|--------|
| 线程安全 | ❌ 单线程 | ✅ 多线程 |
| 性能 | 更快 | 原子操作有开销 |
| 位置 | `std::rc::Rc` | `std::sync::Arc` |

**原则：单线程用 Rc，多线程用 Arc**

如果你用 Rc 跨线程，编译器直接报错：
```
error: `Rc<Vec<i32>>` cannot be sent between threads safely
```

---

## 循环引用问题

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
    next: Option<Rc<RefCell<Node>>>,
}

fn main() {
    let a = Rc::new(RefCell::new(Node { next: None }));
    let b = Rc::new(RefCell::new(Node { next: Some(Rc::clone(&a)) }));
    
    // 让 a 指向 b，形成循环！
    a.borrow_mut().next = Some(Rc::clone(&b));
    
    // a -> b -> a -> b -> ...
    // 引用计数永远不会归零
    // 💀 内存泄漏！
}
```

---

## Weak<T> — 打破循环

`Weak` 是弱引用，不增加强引用计数：

```rust
use std::rc::{Rc, Weak};

struct Node {
    value: i32,
    parent: Option<Weak<Node>>,  // 用 Weak 指向父节点
    children: Vec<Rc<Node>>,     // 用 Rc 拥有子节点
}
```

**Weak 的特点：**
- 不阻止数据释放
- 使用前需要 `upgrade()` 检查是否还存活

```rust
let weak: Weak<String> = /* ... */;

match weak.upgrade() {
    Some(rc) => println!("还活着: {}", rc),
    None => println!("已经被释放了"),
}
```

---

## 实战：共享配置

```rust
use std::sync::Arc;
use std::thread;

struct Config {
    api_url: String,
    timeout: u64,
}

fn main() {
    let config = Arc::new(Config {
        api_url: String::from("https://api.example.com"),
        timeout: 30,
    });
    
    let mut handles = vec![];
    
    for i in 0..3 {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            println!("线程 {} 使用: {}", i, config.api_url);
        }));
    }
    
    for h in handles {
        h.join().unwrap();
    }
}
```

---

## 对比 PHP

PHP 对象默认是引用传递：

```php
$config = new Config();
$a = $config;
$b = $config;
// $a, $b, $config 指向同一个对象
// GC 会在没有引用时清理
```

Rust 需要显式使用 `Rc`/`Arc`：
- **明确意图**：知道这是共享所有权
- **编译时检查**：线程安全由类型系统保证
- **无 GC 开销**：计数归零立即释放

---

## 要点总结

| 类型 | 用途 | 特点 |
|------|------|------|
| `Rc<T>` | 单线程共享 | 引用计数，不可变 |
| `Arc<T>` | 多线程共享 | 原子计数，不可变 |
| `Weak<T>` | 弱引用 | 不增加计数，防止循环引用 |

**记住：**
- 单线程 → `Rc`
- 多线程 → `Arc`
- 父子循环 → 父用 `Rc`，子用 `Weak` 指向父

---

## 下节预告

**RefCell 与内部可变性**

`Rc<T>` 只给不可变引用，如果我就是要修改呢？

---

*课程笔记：性奴001*
