# 第 79 课：std::mem — 内存操作工具箱

> 日期：2026-02-22  
> 主题：std::mem 模块的常用函数

---

## 概述

`std::mem` 是标准库里一个非常实用的模块，提供了一系列底层内存操作的工具函数，是写高效 Rust 代码的必备知识。

---

## 📏 获取类型信息

### `size_of` — 类型大小

```rust
use std::mem;

// 获取类型占用的字节数
println!("u8:  {} bytes", mem::size_of::<u8>());   // 1
println!("u32: {} bytes", mem::size_of::<u32>());  // 4
println!("u64: {} bytes", mem::size_of::<u64>());  // 8

// 引用大小（指针宽度）
println!("&str: {} bytes", mem::size_of::<&str>()); // 16 (胖指针：ptr + len)
println!("&[u8]: {} bytes", mem::size_of::<&[u8]>()); // 16

// Option 的神奇优化
println!("Option<&str>: {} bytes", mem::size_of::<Option<&str>>()); // 16!
// 利用 null pointer optimization，Option<&T> 和 &T 大小相同
```

### `align_of` — 内存对齐

```rust
// 每种类型都有对齐要求
println!("u8 align:  {}", mem::align_of::<u8>());   // 1
println!("u32 align: {}", mem::align_of::<u32>());  // 4
println!("u64 align: {}", mem::align_of::<u64>());  // 8

// struct 的对齐 = 最大字段的对齐
struct Example {
    a: u8,   // 1 byte
    b: u32,  // 4 bytes, 需要 4 字节对齐
}
println!("Example align: {}", mem::align_of::<Example>()); // 4
println!("Example size: {}", mem::size_of::<Example>());   // 8 (有 padding)
```

**对比 PHP/JS：** 这些语言隐藏了内存布局，你根本不需要关心。Rust 暴露这些是为了让你写高性能代码。

---

## 🔄 swap — 交换两个值

```rust
let mut a = String::from("hello");
let mut b = String::from("world");

mem::swap(&mut a, &mut b);

println!("a = {}", a); // world
println!("b = {}", b); // hello
```

**为什么需要 swap？**

在 PHP 里你可能这样写：
```php
$temp = $a;
$a = $b;
$b = $temp;
```

Rust 的 `mem::swap` 更高效 —— 它直接在内存层面交换，不需要额外分配。

**实战用途：** 交换 struct 字段、实现排序算法、双缓冲切换。

---

## 🔁 replace — 替换并返回旧值

```rust
let mut value = String::from("old");

// 放入新值，拿出旧值
let old = mem::replace(&mut value, String::from("new"));

println!("old = {}", old);   // old
println!("value = {}", value); // new
```

**经典场景：** 从 struct 中取走字段

```rust
struct Container {
    data: Option<String>,
}

impl Container {
    // 取走 data，留下 None
    fn take_data(&mut self) -> Option<String> {
        mem::replace(&mut self.data, None)
        // 或者更简洁：self.data.take()
    }
}
```

---

## 📦 take — replace 的简化版

当你想用 **默认值** 替换时：

```rust
let mut name = String::from("Alice");

// 取走值，留下默认值（空字符串）
let taken = mem::take(&mut name);

println!("taken = {}", taken); // Alice
println!("name = '{}'", name); // "" (空)
```

**要求：** 类型必须实现 `Default` trait。

**常见用法：**

```rust
struct State {
    buffer: Vec<u8>,
}

impl State {
    fn flush(&mut self) -> Vec<u8> {
        // 取走 buffer，留下空 Vec
        mem::take(&mut self.buffer)
    }
}
```

---

## 🗑️ drop — 提前释放资源

```rust
let data = vec![1, 2, 3, 4, 5];

// 正常情况：data 在作用域结束时释放
// 但有时你想提前释放

mem::drop(data); // 立即释放！

// println!("{:?}", data); // 编译错误：已被移动
```

**实战场景：**

```rust
use std::sync::Mutex;

let mutex = Mutex::new(42);

{
    let guard = mutex.lock().unwrap();
    println!("locked: {}", *guard);
    
    // 假设这里有很长的操作，不再需要锁...
    mem::drop(guard); // 提前释放锁！
    
    // 其他线程现在可以获取锁了
    do_something_slow();
}
```

---

## 🔬 forget — 阻止析构

```rust
let s = String::from("hello");

mem::forget(s); // s 的内存不会被释放！

// 危险：这会造成内存泄漏！
```

**什么时候用？**

1. 和 C 代码交互，把所有权转移给 C
2. 实现某些 unsafe 的数据结构
3. 创建静态生命周期的值

```rust
// FFI 场景
extern "C" {
    fn c_takes_ownership(ptr: *mut u8, len: usize);
}

let mut data = vec![1u8, 2, 3];
let ptr = data.as_mut_ptr();
let len = data.len();

unsafe {
    c_takes_ownership(ptr, len);
}
mem::forget(data); // 阻止 Rust 释放内存，C 会管理
```

**⚠️ 警告：** 99% 的情况你不需要 `forget`。如果你觉得需要，先想想是不是设计有问题。

---

## 📊 总结

| 函数 | 用途 | 常见场景 |
|------|------|----------|
| `size_of` | 获取类型大小 | 性能优化、序列化 |
| `align_of` | 获取对齐要求 | FFI、自定义分配器 |
| `swap` | 交换两个值 | 排序、状态切换 |
| `replace` | 替换并返回旧值 | 从 struct 取出字段 |
| `take` | 取走值留下默认值 | flush buffer、重置状态 |
| `drop` | 提前释放 | 提前释放锁、文件句柄 |
| `forget` | 阻止析构 | FFI、特殊场景（慎用） |

---

## 💡 最佳实践

```rust
// ✅ 从 Option 取值用 take()
let mut opt = Some(String::from("hello"));
let val = opt.take(); // 比 mem::replace 更优雅

// ✅ 交换字段用 std::mem::swap
mem::swap(&mut self.current, &mut self.previous);

// ✅ 需要提前释放资源用 drop()
let lock = mutex.lock().unwrap();
// ... 使用 lock ...
drop(lock); // 提前释放

// ❌ 除非 FFI，否则不要用 forget
```

---

## 延伸阅读

- [std::mem 文档](https://doc.rust-lang.org/std/mem/)
- [The Rustonomicon - Working with Memory](https://doc.rust-lang.org/nomicon/)
