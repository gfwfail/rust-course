# 第75课：ManuallyDrop — 手动控制析构

> 日期：2026-02-22  
> 主题：用 ManuallyDrop 阻止自动 drop，实现精细的内存控制

---

## 什么是 ManuallyDrop？

在 Rust 中，当值离开作用域时，编译器会**自动调用 drop**。但有时候，我们需要：

- **阻止** 自动析构
- **推迟** 析构到特定时机
- **手动控制** 析构顺序

`ManuallyDrop<T>` 就是干这个的——它是一个**零成本包装器**，告诉编译器："这个值的 drop 我自己来处理！"

```rust
use std::mem::ManuallyDrop;

let mut data = ManuallyDrop::new(String::from("hello"));
// data 离开作用域时，String 不会被自动 drop！
// 内存泄漏！（除非你手动处理）
```

---

## 核心 API

```rust
use std::mem::ManuallyDrop;

// 创建
let md = ManuallyDrop::new(value);

// 获取内部引用
let inner: &T = &*md;
let inner: &mut T = &mut *md;

// 取出值（消耗包装器）
let value: T = ManuallyDrop::into_inner(md);

// 手动调用 drop（unsafe！）
unsafe {
    ManuallyDrop::drop(&mut md);
}

// 取出值但不 drop 包装器（unsafe！）
let value: T = unsafe { ManuallyDrop::take(&mut md) };
```

---

## 实战场景

### 场景1：避免双重释放

当你需要转移所有权但不想触发 drop 时：

```rust
use std::mem::ManuallyDrop;

struct Buffer {
    ptr: *mut u8,
    len: usize,
}

impl Buffer {
    // 把 Buffer 转换成 Vec，转移所有权
    fn into_vec(self) -> Vec<u8> {
        // 用 ManuallyDrop 阻止 Buffer 的 drop
        let md = ManuallyDrop::new(self);
        
        // 安全地重建 Vec
        unsafe {
            Vec::from_raw_parts(md.ptr, md.len, md.len)
        }
        // md 不会被 drop，所以 ptr 只被 Vec 管理
        // 避免了双重释放！
    }
}

impl Drop for Buffer {
    fn drop(&mut self) {
        // 释放内存...
    }
}
```

### 场景2：联合体中的 Drop 类型

Rust 的 `union` 不能自动 drop 字段，需要 ManuallyDrop：

```rust
use std::mem::ManuallyDrop;

union MyUnion {
    // union 字段如果需要 Drop，必须用 ManuallyDrop
    s: ManuallyDrop<String>,
    n: u64,
}

fn main() {
    let mut u = MyUnion {
        s: ManuallyDrop::new(String::from("hello"))
    };
    
    // 读取前需要 unsafe
    unsafe {
        println!("{}", &*u.s);
        
        // 切换到数字前，先手动 drop 字符串
        ManuallyDrop::drop(&mut u.s);
        u.n = 42;
    }
}
```

### 场景3：自定义析构顺序

有时需要精确控制多个值的析构顺序：

```rust
use std::mem::ManuallyDrop;

struct Connection { /* ... */ }
struct Transaction { /* ... */ }

fn complex_cleanup(conn: Connection, tx: Transaction) {
    // 正常情况：tx 先 drop，conn 后 drop（声明的逆序）
    // 但如果需要 conn 先 drop 呢？
    
    let conn = ManuallyDrop::new(conn);
    let tx = ManuallyDrop::new(tx);
    
    // 现在可以控制顺序
    unsafe {
        // 先清理连接
        ManuallyDrop::drop(&mut ManuallyDrop::new(
            ManuallyDrop::into_inner(conn)
        ));
        // 再清理事务（实际项目中这可能有特殊原因）
        ManuallyDrop::drop(&mut ManuallyDrop::new(
            ManuallyDrop::into_inner(tx)
        ));
    }
}
```

---

## drop vs ManuallyDrop::drop

```rust
use std::mem::ManuallyDrop;

let s = String::from("hello");

// 方法1：std::mem::drop — 安全，消耗值
std::mem::drop(s);  // s 被移走，不能再用

// 方法2：ManuallyDrop::drop — unsafe，就地析构
let mut md = ManuallyDrop::new(String::from("world"));
unsafe {
    ManuallyDrop::drop(&mut md);
}
// ⚠️ md 仍然存在，但内部值已析构
// 再访问是 UB（未定义行为）！
```

---

## 典型错误

```rust
use std::mem::ManuallyDrop;

// ❌ 错误：忘记手动 drop = 内存泄漏
fn leak() {
    let data = ManuallyDrop::new(vec![1, 2, 3]);
} // Vec 永远不会被释放！

// ❌ 错误：drop 后再访问 = UB
fn use_after_drop() {
    let mut data = ManuallyDrop::new(String::from("oops"));
    unsafe {
        ManuallyDrop::drop(&mut data);
        println!("{}", &*data); // 💥 未定义行为！
    }
}

// ❌ 错误：双重 drop = UB
fn double_drop() {
    let mut data = ManuallyDrop::new(String::from("bad"));
    unsafe {
        ManuallyDrop::drop(&mut data);
        ManuallyDrop::drop(&mut data); // 💥 双重释放！
    }
}
```

---

## 与 MaybeUninit 配合

在底层代码中经常一起使用：

```rust
use std::mem::{ManuallyDrop, MaybeUninit};

// 手动管理数组初始化
fn init_array<T, F>(init: F) -> [T; 4]
where
    F: Fn(usize) -> T,
{
    let mut arr: [MaybeUninit<T>; 4] = unsafe {
        MaybeUninit::uninit().assume_init()
    };
    
    for (i, slot) in arr.iter_mut().enumerate() {
        slot.write(init(i));
    }
    
    // 转换成初始化后的数组
    // 需要 ManuallyDrop 避免 MaybeUninit 被 drop
    let arr = ManuallyDrop::new(arr);
    unsafe {
        std::ptr::read(arr.as_ptr() as *const [T; 4])
    }
}
```

---

## 零成本抽象

ManuallyDrop 是真正的**零成本**：

```rust
use std::mem::{size_of, ManuallyDrop};

assert_eq!(size_of::<String>(), size_of::<ManuallyDrop<String>>());
assert_eq!(size_of::<Vec<u8>>(), size_of::<ManuallyDrop<Vec<u8>>>());

// 编译后，ManuallyDrop 完全消失
// 只是编译期告诉编译器不要插入 drop 代码
```

---

## 实际应用：标准库中的使用

标准库大量使用 ManuallyDrop：

```rust
// Vec::into_raw_parts 的简化实现
impl<T> Vec<T> {
    pub fn into_raw_parts(self) -> (*mut T, usize, usize) {
        let mut me = ManuallyDrop::new(self);
        (me.as_mut_ptr(), me.len(), me.capacity())
        // Vec 不被 drop，指针可以安全转移
    }
}

// Box::leak 的简化实现  
impl<T> Box<T> {
    pub fn leak(b: Box<T>) -> &'static mut T {
        unsafe {
            &mut *ManuallyDrop::new(b).as_mut_ptr()
        }
        // Box 不被 drop，内存永久保留
    }
}
```

---

## 总结

| 功能 | 方法 |
|------|------|
| 创建 | `ManuallyDrop::new(value)` |
| 访问内部 | `&*md` 或 `&mut *md` |
| 取出值 | `ManuallyDrop::into_inner(md)` |
| 手动 drop | `unsafe { ManuallyDrop::drop(&mut md) }` |
| 取出不 drop | `unsafe { ManuallyDrop::take(&mut md) }` |

### 使用场景

1. **FFI** — 把内存所有权转移给 C 代码
2. **union** — 存放需要 Drop 的类型
3. **避免双重释放** — 转换类型时
4. **自定义析构顺序** — 精确控制
5. **性能优化** — 避免不必要的 drop

### ⚠️ 安全警告

- ManuallyDrop 后如果不处理 = **内存泄漏**
- drop 后再访问 = **未定义行为**
- 双重 drop = **未定义行为**

---

## 对比其他语言

| 语言 | 析构控制 |
|------|----------|
| **PHP/JS** | GC 全自动，无法控制 |
| **Java** | finalize 已废弃，用 try-with-resources |
| **C++** | 析构函数，可能意外调用 |
| **Rust** | 编译器自动 + ManuallyDrop 手动控制 |

Rust 给你最大的控制权，同时通过 unsafe 边界保持安全！

---

*下节预告：MaybeUninit — 未初始化内存的安全抽象*
