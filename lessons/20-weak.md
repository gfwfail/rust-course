# 第20课：Weak 与循环引用

## 循环引用问题

`Rc<T>` 有个致命缺陷：**循环引用会导致内存泄漏**！

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
}

fn main() {
    let a = Rc::new(RefCell::new(Node { value: 1, next: None }));
    let b = Rc::new(RefCell::new(Node { value: 2, next: None }));
    
    // a -> b
    a.borrow_mut().next = Some(Rc::clone(&b));
    // b -> a  ❌ 循环引用！
    b.borrow_mut().next = Some(Rc::clone(&a));
    
    // 程序结束时：
    // a 的引用计数 = 2（变量 a + b.next）
    // b 的引用计数 = 2（变量 b + a.next）
    // 都不会归零，永远不会被 drop！💀
}
```

## Weak 登场

`Weak<T>` 是弱引用，**不增加强引用计数**：

```rust
use std::rc::{Rc, Weak};

fn main() {
    let strong = Rc::new(42);
    
    // Rc -> Weak（downgrade 降级）
    let weak: Weak<i32> = Rc::downgrade(&strong);
    
    println!("强引用计数: {}", Rc::strong_count(&strong)); // 1
    println!("弱引用计数: {}", Rc::weak_count(&strong));   // 1
    
    // Weak -> Rc（upgrade 升级，可能失败）
    if let Some(rc) = weak.upgrade() {
        println!("值还在: {}", rc);
    }
}
```

## 关键区别

| | Rc<T> | Weak<T> |
|---|---|---|
| 影响内存释放 | ✅ 是 | ❌ 否 |
| 保证数据存活 | ✅ 是 | ❌ 否 |
| 访问数据 | 直接访问 | 需 upgrade() |

## upgrade() 可能失败

```rust
use std::rc::{Rc, Weak};

fn main() {
    let weak: Weak<i32>;
    
    {
        let strong = Rc::new(100);
        weak = Rc::downgrade(&strong);
        
        // 这里 upgrade 成功
        assert!(weak.upgrade().is_some());
    } // strong 离开作用域，数据被释放
    
    // 这里 upgrade 失败，返回 None
    assert!(weak.upgrade().is_none());
}
```

## 实战：树结构（父子引用）

父节点拥有子节点，子节点弱引用父节点：

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,      // 弱引用！
    children: RefCell<Vec<Rc<Node>>>, // 强引用
}

fn main() {
    let root = Rc::new(Node {
        value: 1,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });
    
    let child = Rc::new(Node {
        value: 2,
        parent: RefCell::new(Rc::downgrade(&root)), // 弱引用父节点
        children: RefCell::new(vec![]),
    });
    
    root.children.borrow_mut().push(Rc::clone(&child));
}
```

**设计原则**：「拥有」用强引用，「访问」用弱引用。

---

**下节课**：线程基础
