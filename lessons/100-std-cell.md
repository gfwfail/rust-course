# 第 100 课：内部可变性（Interior Mutability）

🎉 里程碑课程！

## 问题背景

Rust 的借用规则很严格：
- 要么有多个不可变引用 `&T`
- 要么只有一个可变引用 `&mut T`

但有时候，我们需要在只有 `&T`（不可变引用）的情况下修改数据。这就是内部可变性要解决的问题。

## Cell<T> - 简单值的内部可变性

`Cell<T>` 适用于 `Copy` 类型，通过 get/set 来操作：

```rust
use std::cell::Cell;

struct Counter {
    value: Cell<i32>,  // 即使 Counter 是不可变的，value 也能改
}

fn main() {
    let counter = Counter { value: Cell::new(0) };
    
    // counter 没有声明为 mut，但我们可以修改内部值！
    counter.value.set(1);
    counter.value.set(counter.value.get() + 1);
    
    println!("Count: {}", counter.value.get()); // 2
}
```

**特点：**
- ✅ 零运行时开销
- ✅ 线程安全检查在编译期
- ❌ 只能用于 `Copy` 类型
- ❌ 不能获取内部值的引用

## RefCell<T> - 运行时借用检查

`RefCell<T>` 把借用检查从编译期移到运行时：

```rust
use std::cell::RefCell;

struct Document {
    content: RefCell<String>,
    edit_count: Cell<usize>,
}

impl Document {
    fn append(&self, text: &str) {
        // borrow_mut() 获取可变引用
        self.content.borrow_mut().push_str(text);
        self.edit_count.set(self.edit_count.get() + 1);
    }
    
    fn read(&self) -> String {
        // borrow() 获取不可变引用
        self.content.borrow().clone()
    }
}

fn main() {
    let doc = Document {
        content: RefCell::new(String::from("Hello")),
        edit_count: Cell::new(0),
    };
    
    doc.append(", World!");
    doc.append(" 🦀");
    
    println!("{}", doc.read());       // Hello, World! 🦀
    println!("Edits: {}", doc.edit_count.get()); // 2
}
```

## RefCell 的运行时 panic

如果违反借用规则，RefCell 会在**运行时** panic：

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);
    
    let borrow1 = data.borrow();     // 不可变借用
    let borrow2 = data.borrow_mut(); // 💥 panic! 已经有不可变借用了
}
```

**安全的做法**：使用 `try_borrow()` / `try_borrow_mut()`：

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(42);
    
    let _r1 = data.borrow();
    
    match data.try_borrow_mut() {
        Ok(mut val) => *val += 1,
        Err(_) => println!("无法获取可变借用"),
    }
}
```

## 实际应用：图结构中的共享可变

```rust
use std::cell::RefCell;
use std::rc::Rc;

type NodeRef = Rc<RefCell<Node>>;

struct Node {
    value: i32,
    neighbors: Vec<NodeRef>,
}

fn main() {
    let node_a = Rc::new(RefCell::new(Node {
        value: 1,
        neighbors: vec![],
    }));
    
    let node_b = Rc::new(RefCell::new(Node {
        value: 2,
        neighbors: vec![Rc::clone(&node_a)], // B -> A
    }));
    
    // 给 A 添加到 B 的连接，形成双向
    node_a.borrow_mut().neighbors.push(Rc::clone(&node_b));
    
    println!("A 的邻居数: {}", node_a.borrow().neighbors.len()); // 1
    println!("B 的邻居数: {}", node_b.borrow().neighbors.len()); // 1
}
```

`Rc<RefCell<T>>` 是单线程共享可变状态的经典组合。

## Cell vs RefCell 对比

| 特性 | Cell<T> | RefCell<T> |
|------|---------|------------|
| 适用类型 | Copy 类型 | 任意类型 |
| 获取引用 | ❌ 不能 | ✅ 可以 |
| 性能 | 零开销 | 有运行时检查 |
| 失败行为 | 编译错误 | 运行时 panic |

## 为什么 Rust 需要内部可变性？

1. **共享所有权场景**：`Rc<T>` 只能获得不可变引用，配合 `RefCell` 才能修改
2. **回调/闭包**：闭包捕获的变量需要修改
3. **缓存/记忆化**：在不可变接口下缓存计算结果
4. **GUI/游戏开发**：多个组件共享修改状态

## 黄金法则

> 编译器无法证明借用安全时，用 `RefCell` 把检查推迟到运行时。
> 但你需要**自己保证**不会违反借用规则。

---

下节课：OnceCell 和 Lazy —— 延迟初始化的利器！
