# 第 37 课：Unsafe Rust

> 📅 2026-02-17 09:00 (AEDT)

---

## 上节回顾

上节课学了过程宏 —— 编译期操作 AST，自动生成代码。

今天进入 Rust 的「禁区」：**Unsafe Rust**。

---

## 为什么需要 unsafe？

Rust 的安全保证很强，但有些事它做不到：

```rust
// ❌ Rust 编译器无法验证
// 1. 操作硬件
// 2. 调用 C 库
// 3. 实现某些数据结构（链表、图）
// 4. 极致性能优化
```

`unsafe` 是你告诉编译器："相信我，我知道自己在做什么。"

---

## unsafe 能做什么？（5 种超能力）

```rust
unsafe {
    // 1. 解引用裸指针
    // 2. 调用 unsafe 函数
    // 3. 访问/修改可变静态变量
    // 4. 实现 unsafe trait
    // 5. 访问 union 的字段
}
```

**重要：** unsafe 不会关闭借用检查！它只解锁这 5 种操作。

---

## 1. 裸指针 (Raw Pointers)

Rust 有两种裸指针：
- `*const T` — 不可变裸指针
- `*mut T` — 可变裸指针

```rust
fn main() {
    let mut x = 10;
    
    // 创建裸指针是安全的
    let ptr_const: *const i32 = &x;
    let ptr_mut: *mut i32 = &mut x;
    
    // ⚠️ 解引用裸指针需要 unsafe！
    unsafe {
        println!("值: {}", *ptr_const);
        *ptr_mut = 20;
        println!("新值: {}", *ptr_mut);
    }
}
```

**裸指针 vs 引用：**

| 特性 | 引用 &T | 裸指针 *const T |
|------|--------|-----------------|
| 可以为空 | ❌ | ✅ |
| 自动释放 | ✅ | ❌ |
| 借用检查 | ✅ | ❌ |
| 可以悬垂 | ❌ | ✅ |

---

## 2. unsafe 函数

函数本身可以标记为 unsafe：

```rust
// 声明 unsafe 函数
unsafe fn dangerous() {
    // 函数体自动处于 unsafe 上下文
}

fn main() {
    // 调用时必须用 unsafe 块
    unsafe {
        dangerous();
    }
}
```

**实际例子：** `slice::get_unchecked`

```rust
fn main() {
    let arr = [1, 2, 3, 4, 5];
    
    // 安全版本：有边界检查
    let val = arr.get(10);  // 返回 None
    
    // unsafe 版本：跳过边界检查
    unsafe {
        let val = arr.get_unchecked(2);  // 直接返回 &3
        // arr.get_unchecked(10);  // 未定义行为！💀
    }
}
```

---

## 3. 可变静态变量

全局可变状态在多线程下是危险的：

```rust
static mut COUNTER: i32 = 0;

fn increment() {
    unsafe {
        COUNTER += 1;  // ⚠️ 非线程安全！
    }
}

fn main() {
    unsafe {
        COUNTER = 10;
        println!("Counter: {}", COUNTER);
    }
}
```

**更好的选择：** 用 `AtomicI32` 或 `Mutex`

```rust
use std::sync::atomic::{AtomicI32, Ordering};

static COUNTER: AtomicI32 = AtomicI32::new(0);

fn increment() {
    COUNTER.fetch_add(1, Ordering::SeqCst);  // 安全！
}
```

---

## 4. unsafe trait

有些 trait 的实现者必须保证特定不变量：

```rust
// Send: 可以跨线程移动
// Sync: 可以跨线程共享
// 这两个是 unsafe trait！

struct MyType {
    ptr: *mut i32,
}

// 实现 unsafe trait 需要 unsafe impl
unsafe impl Send for MyType {}
unsafe impl Sync for MyType {}
```

**警告：** 除非你完全理解 Send/Sync 的语义，否则不要手动实现！

---

## 5. 访问 Union 字段

Union 类似 C 的 union，多个字段共享内存：

```rust
union IntOrFloat {
    i: i32,
    f: f32,
}

fn main() {
    let u = IntOrFloat { i: 42 };
    
    unsafe {
        println!("作为 int: {}", u.i);
        println!("作为 float: {}", u.f);  // 重新解释比特位
    }
}
```

---

## 安全抽象：unsafe 的正确用法

**核心理念：** 在 unsafe 代码外面包一层安全的 API。

标准库经典例子：`Vec`

```rust
// Vec 内部用了大量 unsafe
// 但对外暴露的是完全安全的 API

let mut v = vec![1, 2, 3];
v.push(4);      // 安全
v.pop();        // 安全
let x = v[0];   // 安全（有边界检查）
```

**自己实现安全抽象：**

```rust
fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();
    
    assert!(mid <= len);  // 先做安全检查！
    
    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut arr = [1, 2, 3, 4, 5];
    let (left, right) = split_at_mut(&mut arr, 3);
    // left: [1, 2, 3], right: [4, 5]
}
```

**关键点：**
1. 外部函数是安全的（没有 `unsafe fn`）
2. 在调用 unsafe 代码前做好检查 (`assert!`)
3. unsafe 块尽可能小

---

## FFI：调用 C 代码

Foreign Function Interface —— 最常见的 unsafe 用途：

```rust
// 声明外部 C 函数
extern "C" {
    fn abs(input: i32) -> i32;
    fn strlen(s: *const i8) -> usize;
}

fn main() {
    unsafe {
        println!("abs(-5) = {}", abs(-5));
    }
}
```

**调用系统库：**

```rust
use std::ffi::CString;

extern "C" {
    fn printf(format: *const i8, ...) -> i32;
}

fn main() {
    let msg = CString::new("Hello from Rust! %d\n").unwrap();
    
    unsafe {
        printf(msg.as_ptr(), 42);
    }
}
```

**让 Rust 函数可被 C 调用：**

```rust
#[no_mangle]
pub extern "C" fn rust_function(x: i32) -> i32 {
    x * 2
}
```

---

## 实战：用裸指针实现简单链表节点

```rust
use std::ptr;

struct Node {
    value: i32,
    next: *mut Node,  // 裸指针！
}

impl Node {
    fn new(value: i32) -> *mut Node {
        let node = Box::new(Node {
            value,
            next: ptr::null_mut(),
        });
        Box::into_raw(node)  // 转成裸指针
    }
}

fn main() {
    unsafe {
        // 创建节点
        let node1 = Node::new(1);
        let node2 = Node::new(2);
        let node3 = Node::new(3);
        
        // 链接
        (*node1).next = node2;
        (*node2).next = node3;
        
        // 遍历
        let mut current = node1;
        while !current.is_null() {
            println!("Value: {}", (*current).value);
            current = (*current).next;
        }
        
        // 别忘了释放内存！
        drop(Box::from_raw(node1));
        drop(Box::from_raw(node2));
        drop(Box::from_raw(node3));
    }
}
```

**为什么链表需要 unsafe？**

Rust 的借用规则无法表达「多个节点互相指向」的结构。

实际项目中，推荐用现成的安全实现或 `Rc<RefCell<Node>>`。

---

## unsafe 最佳实践

### ✅ DO

```rust
// 1. unsafe 块要小
unsafe {
    // 只放必须 unsafe 的代码
}

// 2. 写清楚为什么这是安全的
// SAFETY: ptr 在上面已验证非空且指向有效内存
unsafe {
    *ptr = 42;
}

// 3. 暴露安全的 API
pub fn safe_wrapper(...) {
    // 内部 unsafe
}
```

### ❌ DON'T

```rust
// 1. 不要整个函数标记 unsafe（除非必要）
unsafe fn everything_is_unsafe() { ... }  // 太宽泛

// 2. 不要假设"应该没问题"
unsafe {
    // "这个指针肯定有效吧" ← 💀
    *mystery_ptr = 10;
}

// 3. 不要到处 unsafe 来绕过借用检查
```

---

## 什么时候用 unsafe？

| 场景 | 是否需要 unsafe |
|------|-----------------|
| 调用 C 库 | ✅ 是 |
| 极致性能（避免边界检查） | ✅ 是 |
| 实现特殊数据结构 | ✅ 可能 |
| 借用检查很烦 | ❌ 不！重新设计！ |
| 想偷懒 | ❌ 绝对不！ |

**记住：** 95% 的 Rust 代码不需要 unsafe。如果你发现自己频繁需要 unsafe，大概率是设计问题。

---

## 常见的 unsafe 代码/库

- `std::ptr` — 裸指针操作
- `std::mem` — 内存操作 (transmute, forget)
- `std::slice::from_raw_parts` — 从裸指针创建 slice
- `libc` — C 标准库绑定
- `winapi` / `windows` — Windows API 绑定
- `nix` — Unix API 绑定

---

## 课后练习

尝试实现一个简单的 unsafe 函数，交换两个变量的值：

```rust
unsafe fn swap_raw<T>(a: *mut T, b: *mut T) {
    // 你的实现
}

fn main() {
    let mut x = 1;
    let mut y = 2;
    
    unsafe {
        swap_raw(&mut x, &mut y);
    }
    
    assert_eq!(x, 2);
    assert_eq!(y, 1);
}
```

---

## 下节预告

下节课：**Cargo 深入 —— Workspace、Features 与条件编译**
- 多 crate 项目管理
- feature flags
- 条件编译 cfg

---

*笔记整理：性奴001*
