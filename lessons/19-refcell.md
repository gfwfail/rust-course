# 第19课：RefCell 内部可变性

普通 Rust：编译时检查借用规则。
`RefCell`：**运行时检查**，允许"看起来不可变，实际能改"。

## 基本用法

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(5);
    
    // 运行时借用检查
    *data.borrow_mut() += 10;  // 可变借用
    println!("data = {}", data.borrow());  // 不可变借用
}
```

## 借用方法

| 方法 | 返回 | 失败时 |
|-----|-----|-------|
| `borrow()` | `Ref<T>` | panic! |
| `borrow_mut()` | `RefMut<T>` | panic! |
| `try_borrow()` | `Result<Ref<T>>` | 返回 Err |
| `try_borrow_mut()` | `Result<RefMut<T>>` | 返回 Err |

## 借用规则仍然适用

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(5);
    
    let r1 = data.borrow();
    let r2 = data.borrow();      // ✅ 多个不可变借用
    // let r3 = data.borrow_mut();  // ❌ panic! 已有不可变借用
    
    drop(r1);
    drop(r2);
    
    let r3 = data.borrow_mut();  // ✅ 现在可以了
}
```

## Rc + RefCell = 多个所有者 + 可变

单独用 Rc 只能共享不可变数据。配合 RefCell 就能共享可变数据！

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    // 多个所有者，每个都能修改
    let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
    
    let a = Rc::clone(&shared);
    let b = Rc::clone(&shared);
    
    a.borrow_mut().push(4);  // a 修改
    b.borrow_mut().push(5);  // b 修改
    
    println!("{:?}", shared.borrow());  // [1, 2, 3, 4, 5]
}
```

## 三者对比

| 类型 | 所有权 | 可变性 | 检查时机 | 线程安全 |
|-----|-------|-------|---------|---------|
| `Box<T>` | 单一 | 借用规则 | 编译时 | 可Send |
| `Rc<T>` | 多个 | 不可变 | 编译时 | ❌ |
| `RefCell<T>` | 单一 | 内部可变 | 运行时 | ❌ |
| `Rc<RefCell<T>>` | 多个 | 内部可变 | 运行时 | ❌ |

---

**下节课**：Weak 与循环引用
