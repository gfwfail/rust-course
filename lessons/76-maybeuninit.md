# 第 76 课：MaybeUninit — 未初始化内存的安全处理

> 日期：2026-02-22  
> 主题：std::mem::MaybeUninit 的原理与使用

---

## 为什么需要 MaybeUninit？

在 Rust 中，使用未初始化的内存是 **未定义行为（UB）**：

```rust
// ❌ 这是 UB！编译器可能会做任何事
let x: i32;
println!("{}", x);  // 未初始化就使用
```

但有些场景我们确实需要"先分配空间，稍后再填充"：
- 性能优化：避免不必要的初始化
- FFI：C 代码会填充数据
- 数组初始化：逐个填充元素

`MaybeUninit` 就是这个工具。

---

## 基本用法

```rust
use std::mem::MaybeUninit;

// 创建未初始化的内存
let mut x: MaybeUninit<i32> = MaybeUninit::uninit();

// 写入值
x.write(42);

// 安全地获取值（你保证已经初始化）
let value = unsafe { x.assume_init() };
println!("{}", value);  // 42
```

**关键点**：
- `MaybeUninit::uninit()` — 创建未初始化内存
- `.write(value)` — 写入值
- `.assume_init()` — **unsafe**，告诉编译器"我保证已初始化"

---

## 对比 PHP/JS

```php
// PHP - 随便用未初始化变量
$x;  // null
echo $x;  // 输出空，不会崩
```

```javascript
// JS - 也是
let x;
console.log(x);  // undefined
```

```rust
// Rust - 严格得多
let x: i32;
// println!("{}", x);  // 编译错误！

// 用 MaybeUninit 显式处理
let mut x = MaybeUninit::<i32>::uninit();
// 这是合法的，因为我们明确说"这是未初始化的"
```

---

## 实际场景：数组初始化

假设要创建一个大数组，逐个计算填充：

```rust
use std::mem::MaybeUninit;

// 方法 1: 传统方式（会先初始化为 0，再覆盖）
let mut arr = [0i32; 1000];
for i in 0..1000 {
    arr[i] = expensive_computation(i);
}

// 方法 2: MaybeUninit（跳过初始化，直接填充）
let arr: [i32; 1000] = {
    // 创建未初始化数组
    let mut arr: [MaybeUninit<i32>; 1000] = 
        unsafe { MaybeUninit::uninit().assume_init() };
    
    // 逐个填充
    for i in 0..1000 {
        arr[i].write(expensive_computation(i));
    }
    
    // 转换为初始化的数组
    unsafe { 
        std::mem::transmute::<_, [i32; 1000]>(arr) 
    }
};
```

现代 Rust 有更简洁的写法：

```rust
use std::mem::MaybeUninit;

let arr: [i32; 1000] = {
    let mut arr = [const { MaybeUninit::uninit() }; 1000];
    
    for (i, slot) in arr.iter_mut().enumerate() {
        slot.write(expensive_computation(i));
    }
    
    unsafe { MaybeUninit::array_assume_init(arr) }
};
```

---

## 常用方法

```rust
use std::mem::MaybeUninit;

let mut x = MaybeUninit::<String>::uninit();

// 写入值
x.write(String::from("hello"));

// 获取可变引用（需要先初始化）
let s = unsafe { x.assume_init_mut() };
s.push_str(" world");

// 读取值（消耗 MaybeUninit）
let s = unsafe { x.assume_init() };
println!("{}", s);  // "hello world"
```

**其他有用方法：**

```rust
// 直接创建已初始化的值
let x = MaybeUninit::new(42);
let value = unsafe { x.assume_init() };

// 获取裸指针
let mut x = MaybeUninit::<i32>::uninit();
let ptr = x.as_mut_ptr();
unsafe { ptr.write(42) };

// 清零（但仍未"初始化"）
let x = MaybeUninit::<i32>::zeroed();
let value = unsafe { x.assume_init() };  // 0
```

---

## 危险操作 — Drop 的陷阱

```rust
use std::mem::MaybeUninit;

// ❌ 危险！如果类型有 Drop，会出问题
let mut x = MaybeUninit::<String>::uninit();
x.write(String::from("hello"));
// 如果这里 x 被丢弃而没有 assume_init()
// String 不会被正确释放！

// ✅ 正确做法
let mut x = MaybeUninit::<String>::uninit();
x.write(String::from("hello"));
let s = unsafe { x.assume_init() };  // 现在 s 负责释放
drop(s);  // String 被正确释放
```

**记住**：`MaybeUninit` 的 Drop 什么都不做。你要自己管理内部值的生命周期。

---

## 与 ManuallyDrop 的关系

| 类型 | 用途 | Drop 行为 |
|------|------|----------|
| `MaybeUninit<T>` | 处理可能未初始化的内存 | 不调用内部 Drop |
| `ManuallyDrop<T>` | 阻止自动 Drop | 不调用内部 Drop |

两者经常配合使用：

```rust
use std::mem::{MaybeUninit, ManuallyDrop};

// MaybeUninit 用于创建
let mut x = MaybeUninit::<String>::uninit();
x.write(String::from("hello"));

// ManuallyDrop 用于取出后手动管理
let s = ManuallyDrop::new(unsafe { x.assume_init() });
// 现在 s 不会自动释放
```

---

## 实际应用：FFI 场景

C 函数经常要求你传入一个缓冲区让它填充：

```rust
use std::mem::MaybeUninit;

// 假设这是 C 函数签名
extern "C" {
    fn fill_buffer(buf: *mut u8, len: usize) -> i32;
}

fn get_data() -> Vec<u8> {
    let mut buffer: [MaybeUninit<u8>; 1024] = 
        [const { MaybeUninit::uninit() }; 1024];
    
    let result = unsafe {
        fill_buffer(
            buffer.as_mut_ptr() as *mut u8,
            buffer.len()
        )
    };
    
    if result > 0 {
        // C 函数填充了 result 字节
        let filled = &buffer[..result as usize];
        unsafe {
            filled.iter()
                .map(|x| x.assume_init_read())
                .collect()
        }
    } else {
        Vec::new()
    }
}
```

---

## 总结

| 概念 | 说明 |
|------|------|
| `MaybeUninit::uninit()` | 创建未初始化内存 |
| `.write(value)` | 安全地写入值 |
| `.assume_init()` | unsafe，声明已初始化并取出 |
| `.assume_init_mut()` | unsafe，获取已初始化值的可变引用 |
| `.assume_init_read()` | unsafe，读取值但不消耗 MaybeUninit |
| 不会调用 Drop | 你要自己管理内部值的生命周期 |

**使用场景**：
- 性能优化（避免无意义的初始化）
- FFI（与 C 代码交互）
- 实现数据结构（如 `Vec` 的内部实现）

**黄金法则**：只在真正需要时使用，优先用安全的初始化方式。

---

*下节课：NonNull — 非空裸指针的安全封装*
