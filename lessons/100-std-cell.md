# 第 100 课：std::cell — 内部可变性（Interior Mutability）

🎉 恭喜！这是我们的第 100 课！

上节课讲了多线程同步的 `std::sync`，今天讲单线程场景下的"内部可变性"——在持有不可变引用时修改数据。

---

## 🤔 为什么需要内部可变性？

Rust 的借用规则很严格：

```rust
// ❌ 不能在有不可变引用时修改
let mut x = 5;
let r = &x;      // 不可变借用
x = 6;           // 编译错误！
println!("{}", r);
```

但有时候我们需要：
- 通过 `&self` 修改内部数据（比如缓存）
- 共享所有权的同时允许修改

---

## 📦 Cell<T> — 值的移动/复制

`Cell<T>` 适用于实现 `Copy` 的类型：

```rust
use std::cell::Cell;

let c = Cell::new(5);

// 注意：c 不需要是 mut！
c.set(10);           // 修改值
let v = c.get();     // 获取值的拷贝
println!("{}", v);   // 10

// 常见用途：计数器
struct Counter {
    count: Cell<i32>,
}

impl Counter {
    fn new() -> Self {
        Counter { count: Cell::new(0) }
    }
    
    // 注意是 &self，不是 &mut self！
    fn increment(&self) {
        self.count.set(self.count.get() + 1);
    }
    
    fn get(&self) -> i32 {
        self.count.get()
    }
}

let counter = Counter::new();
counter.increment();  // 不需要 mut 绑定
counter.increment();
println!("Count: {}", counter.get());  // 2
```

### Cell 的限制

```rust
// Cell 只能整体替换值，不能获取引用
let c = Cell::new(String::from("hello"));
// c.get();  // ❌ String 不是 Copy

// 只能这样：
let old = c.replace(String::from("world"));
let taken = c.take();  // 取出值，Cell 变成 Default
```

---

## 📦 RefCell<T> — 运行时借用检查

`RefCell<T>` 更灵活，可以获取引用：

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// 获取不可变引用
{
    let r = data.borrow();  // Ref<Vec<i32>>
    println!("{:?}", *r);   // [1, 2, 3]
}  // r 离开作用域，借用结束

// 获取可变引用
{
    let mut r = data.borrow_mut();  // RefMut<Vec<i32>>
    r.push(4);
}

println!("{:?}", data.borrow());  // [1, 2, 3, 4]
```

### ⚠️ 运行时 panic！

```rust
use std::cell::RefCell;

let data = RefCell::new(5);

let r1 = data.borrow();      // 不可变借用
let r2 = data.borrow();      // ✅ 多个不可变借用 OK

// let r3 = data.borrow_mut();  // ❌ panic! 已有不可变借用

drop(r1);
drop(r2);
let r3 = data.borrow_mut();  // ✅ 现在可以了
```

借用规则在**运行时**检查，违反就 panic：
- 多个 `borrow()` → OK
- `borrow()` + `borrow_mut()` → panic!
- 多个 `borrow_mut()` → panic!

### try_borrow：避免 panic

```rust
use std::cell::RefCell;

let data = RefCell::new(5);
let r = data.borrow();

// 返回 Result，不会 panic
match data.try_borrow_mut() {
    Ok(mut r) => *r = 10,
    Err(_) => println!("借用冲突！"),
}
```

---

## 🔄 经典模式：Rc<RefCell<T>>

单线程下的共享可变数据：

```rust
use std::rc::Rc;
use std::cell::RefCell;

// 多个所有者 + 可变
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

let a = Rc::clone(&shared);
let b = Rc::clone(&shared);

// a 和 b 都可以修改
a.borrow_mut().push(4);
b.borrow_mut().push(5);

println!("{:?}", shared.borrow());  // [1, 2, 3, 4, 5]
```

### 实际例子：树结构的父节点引用

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,      // 父节点（弱引用）
    children: RefCell<Vec<Rc<Node>>>, // 子节点
}

impl Node {
    fn new(value: i32) -> Rc<Self> {
        Rc::new(Node {
            value,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![]),
        })
    }
    
    fn add_child(parent: &Rc<Node>, child: &Rc<Node>) {
        // 设置子节点的 parent
        *child.parent.borrow_mut() = Rc::downgrade(parent);
        // 添加到父节点的 children
        parent.children.borrow_mut().push(Rc::clone(child));
    }
}

let root = Node::new(1);
let child = Node::new(2);
Node::add_child(&root, &child);

// 从子节点访问父节点
if let Some(p) = child.parent.borrow().upgrade() {
    println!("Parent value: {}", p.value);  // 1
}
```

---

## 📊 Cell vs RefCell vs Mutex

| 特性 | Cell<T> | RefCell<T> | Mutex<T> |
|------|---------|------------|----------|
| 获取引用 | ❌ (只能 get/set) | ✅ | ✅ |
| T 要求 | Copy (或用 take/replace) | 无 | 无 |
| 检查时机 | 无需检查 | 运行时 | 运行时 |
| 线程安全 | ❌ | ❌ | ✅ |
| 开销 | 零 | 很小 | 有锁开销 |

**选择原则**：
- 单线程 + Copy 类型 → `Cell`
- 单线程 + 需要引用 → `RefCell`
- 多线程 → `Mutex` / `RwLock`

---

## 💡 OnceCell — 延迟初始化

```rust
use std::cell::OnceCell;

let cell: OnceCell<String> = OnceCell::new();

// 只能设置一次
assert!(cell.set("hello".to_string()).is_ok());
assert!(cell.set("world".to_string()).is_err());  // 失败

// get_or_init：获取或初始化
let value = cell.get_or_init(|| {
    println!("初始化！");
    "default".to_string()
});
// 不会打印，因为已经有值了

println!("{}", cell.get().unwrap());  // hello
```

常见用途：全局懒加载（配合 `thread_local!` 或 `std::sync::OnceLock`）

---

## 🧠 理解内部可变性的本质

```rust
// 这是编译时借用检查
fn compile_time(x: &mut i32) {
    *x = 10;
}

// 这是运行时借用检查
fn runtime(x: &RefCell<i32>) {
    *x.borrow_mut() = 10;
}
```

`RefCell` 把编译时的检查推迟到运行时，代价是：
1. 微小的运行时开销（引用计数）
2. 可能在运行时 panic

**核心原则**：能用编译时检查就用编译时检查，`Cell`/`RefCell` 是最后手段。

---

## 🎓 课后思考

1. 为什么 `Cell<T>` 要求 `T: Copy`？如果不是 Copy 类型怎么办？
2. `RefCell` 和 `Mutex` 的借用检查有什么本质区别？
3. 什么场景下必须用 `RefCell` 而不能用 `&mut`？

---

*🎉 第 100 课完！我们已经走过了 100 课的旅程！*
