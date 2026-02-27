# 第 100 课：Unsafe Rust — 突破安全边界

恭喜大家！我们迎来了第 100 课的里程碑！今天讲一个重要的话题：**Unsafe Rust**。

---

## 🤔 为什么需要 Unsafe？

Rust 的安全保证很强大，但有些操作编译器无法验证其安全性：

- 直接操作硬件（操作系统、嵌入式）
- 调用 C 语言库（FFI）
- 实现高性能数据结构
- 标准库底层实现

**Unsafe 不是"不安全"的代码，而是"需要程序员保证安全"的代码。**

---

## 📦 Unsafe 的 5 种超能力

```rust
unsafe {
    // 1. 解引用裸指针
    // 2. 调用 unsafe 函数
    // 3. 访问/修改可变静态变量
    // 4. 实现 unsafe trait
    // 5. 访问 union 的字段
}
```

其他操作（如整数溢出、数组越界）仍由编译器检查！

---

## 🔧 超能力一：裸指针

```rust
fn main() {
    let mut num = 5;
    
    // 创建裸指针是安全的
    let r1 = &num as *const i32;      // 不可变裸指针
    let r2 = &mut num as *mut i32;    // 可变裸指针
    
    // 解引用需要 unsafe
    unsafe {
        println!("r1 = {}", *r1);  // 5
        *r2 = 10;
        println!("r2 = {}", *r2);  // 10
    }
}
```

### 裸指针 vs 引用

| 特性 | 引用 `&T` | 裸指针 `*const T` |
|------|-----------|-------------------|
| 保证有效 | ✅ | ❌ |
| 保证对齐 | ✅ | ❌ |
| 可以为 null | ❌ | ✅ |
| 多个可变指针 | ❌ | ✅ |
| 生命周期检查 | ✅ | ❌ |

---

## 🔧 超能力二：unsafe 函数

```rust
// 声明 unsafe 函数
unsafe fn dangerous() {
    println!("我是 unsafe 函数");
}

fn main() {
    // 调用需要 unsafe 块
    unsafe {
        dangerous();
    }
}
```

### 实际例子：slice::from_raw_parts

```rust
use std::slice;

fn main() {
    let arr = [1, 2, 3, 4, 5];
    let ptr = arr.as_ptr();
    
    // 从裸指针创建 slice
    // 程序员必须保证：ptr 有效、长度正确、数据不会被修改
    let slice = unsafe {
        slice::from_raw_parts(ptr, 3)
    };
    
    println!("{:?}", slice);  // [1, 2, 3]
}
```

---

## 🔧 超能力三：可变静态变量

```rust
// 不可变静态变量，安全
static HELLO: &str = "Hello";

// 可变静态变量，危险！
static mut COUNTER: u32 = 0;

fn add_to_counter(val: u32) {
    unsafe {
        COUNTER += val;  // 需要 unsafe
    }
}

fn main() {
    add_to_counter(3);
    
    unsafe {
        println!("COUNTER = {}", COUNTER);  // 3
    }
}
```

⚠️ **为什么危险？** 多线程同时修改会导致数据竞争。优先用 `Mutex` 或 `AtomicU32`！

---

## 🔧 超能力四：unsafe trait

```rust
// 声明 unsafe trait
unsafe trait Dangerous {
    fn do_something(&self);
}

// 实现需要 unsafe impl
unsafe impl Dangerous for i32 {
    fn do_something(&self) {
        println!("i32: {}", self);
    }
}
```

标准库例子：`Send` 和 `Sync` 是 unsafe trait，因为错误实现会导致数据竞争。

---

## 🔧 超能力五：访问 union 字段

```rust
// union 类似 C 的 union，所有字段共享内存
union MyUnion {
    i: i32,
    f: f32,
}

fn main() {
    let u = MyUnion { i: 42 };
    
    // 读取 union 字段需要 unsafe
    // 因为编译器不知道当前存的是哪个类型
    unsafe {
        println!("as i32: {}", u.i);
        println!("as f32: {}", u.f);  // 垃圾值！
    }
}
```

---

## 💡 安全抽象：包装 unsafe

最佳实践：用**安全的 API 封装 unsafe 代码**

```rust
// 不安全的底层操作
fn split_at_mut_manual(slice: &mut [i32], mid: usize) 
    -> (&mut [i32], &mut [i32]) 
{
    let len = slice.len();
    let ptr = slice.as_mut_ptr();
    
    assert!(mid <= len);  // 安全检查！
    
    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut arr = [1, 2, 3, 4, 5];
    
    // 调用者使用安全的 API
    let (left, right) = split_at_mut_manual(&mut arr, 2);
    
    println!("{:?}", left);   // [1, 2]
    println!("{:?}", right);  // [3, 4, 5]
}
```

**这就是标准库 `split_at_mut` 的原理！**

---

## 🌍 FFI：调用 C 语言

```rust
// 声明外部 C 函数
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    let x = -5;
    
    // 调用 C 函数需要 unsafe
    let result = unsafe { abs(x) };
    
    println!("abs({}) = {}", x, result);  // abs(-5) = 5
}
```

### 暴露 Rust 函数给 C

```rust
// no_mangle 防止编译器改名
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}
```

---

## ⚠️ Unsafe 的原则

1. **最小化 unsafe 范围** — 只包裹必要的代码
2. **封装成安全 API** — 内部 unsafe，外部安全
3. **写好注释** — 说明为什么是安全的
4. **加断言** — 用 `assert!` 验证前置条件

```rust
/// 安全性保证：
/// - ptr 必须指向有效的 [T; len] 内存
/// - 内存必须正确初始化
/// - 在 slice 生命周期内不能有其他可变引用
unsafe fn create_slice<'a, T>(ptr: *const T, len: usize) -> &'a [T] {
    // ...
}
```

---

## 🧠 课后思考

1. 为什么 `*const T` 和 `*mut T` 可以同时存在，但 `&T` 和 `&mut T` 不行？
2. 如果要实现一个侵入式链表，为什么需要 unsafe？
3. `Vec<T>` 底层用了哪些 unsafe 操作？

---

🎉 **恭喜完成第 100 课！** 我们从基础语法走到了 Unsafe Rust，这是一个重要的里程碑。接下来会继续深入更多高级主题！

*第 100 课完*
