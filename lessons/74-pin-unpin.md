# 第74课：Pin 与 Unpin — 固定内存位置

> 日期：2026-02-22  
> 主题：理解 Pin 的作用、Unpin trait、以及与异步编程的关系

---

## 为什么需要 Pin？

在 Rust 中，大多数值可以随意移动（move）。但有些情况下，值**不能被移动**，因为它内部包含指向自己的指针（自引用结构）。

最常见的例子：**async/await 生成的 Future**。

```rust
async fn example() {
    let data = vec![1, 2, 3];
    let reference = &data;  // 引用 data
    some_async_work().await; // await 点
    println!("{:?}", reference); // await 后仍然使用 reference
}
```

编译器会把这个 async 块转换成一个状态机结构体，这个结构体**同时持有 `data` 和指向 `data` 的引用**——这就是自引用！

如果这个结构体被移动到内存的另一个位置，那个引用就会指向错误的地址。

---

## Pin 的核心概念

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

// Pin<P> 是一个智能指针包装器
// 它承诺：被包装的值不会被移动

// 两个关键 trait：
// - Unpin: 标记类型可以安全移动（大多数类型都是）
// - !Unpin: 标记类型不能被移动
```

### 类型分类

| 类型 | Unpin? | 可以移动? |
|------|--------|----------|
| `i32`, `String`, `Vec<T>` | ✅ Unpin | ✅ 可以 |
| `async fn` 生成的 Future | ❌ !Unpin | ❌ 不行 |
| 包含 `PhantomPinned` 的类型 | ❌ !Unpin | ❌ 不行 |

---

## 实战示例

### 1. 创建一个自引用结构

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;
use std::ptr::NonNull;

struct SelfRef {
    data: String,
    // 指向 data 的指针
    ptr_to_data: Option<NonNull<String>>,
    // 标记为 !Unpin
    _pin: PhantomPinned,
}

impl SelfRef {
    fn new(data: &str) -> Self {
        SelfRef {
            data: data.to_string(),
            ptr_to_data: None,
            _pin: PhantomPinned,
        }
    }

    // 初始化自引用指针（需要 Pin）
    fn init(self: Pin<&mut Self>) {
        let this = unsafe { self.get_unchecked_mut() };
        let ptr = NonNull::from(&this.data);
        this.ptr_to_data = Some(ptr);
    }

    // 安全地读取数据
    fn get_data(self: Pin<&Self>) -> &str {
        &self.data
    }

    // 通过指针读取（证明指针有效）
    fn get_via_ptr(self: Pin<&Self>) -> &str {
        unsafe {
            self.ptr_to_data
                .map(|p| p.as_ref())
                .unwrap()
        }
    }
}
```

### 2. 使用 Pin

```rust
fn main() {
    // 方法1：使用 Box::pin（堆上固定）
    let mut boxed = Box::pin(SelfRef::new("hello"));
    boxed.as_mut().init();
    
    println!("直接访问: {}", boxed.as_ref().get_data());
    println!("指针访问: {}", boxed.as_ref().get_via_ptr());
    // 两者相同，证明指针有效！

    // 方法2：使用 pin! 宏（栈上固定）
    // Rust 1.68+ 稳定版
    // let mut value = std::pin::pin!(SelfRef::new("world"));
}
```

---

## Pin 的 API

```rust
use std::pin::Pin;

// 创建 Pin
let pinned: Pin<Box<T>> = Box::pin(value);
let pinned: Pin<&mut T> = Pin::new(&mut value); // 仅当 T: Unpin

// 访问内部值
let inner: &T = pinned.as_ref().get_ref();

// 可变访问（如果 T: Unpin）
let inner: &mut T = pinned.as_mut().get_mut();

// 可变访问（如果 T: !Unpin，需要 unsafe）
let inner: &mut T = unsafe { pinned.as_mut().get_unchecked_mut() };
```

---

## 与 Future 的关系

这就是为什么 `Future::poll` 需要 `Pin<&mut Self>`：

```rust
pub trait Future {
    type Output;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) 
        -> Poll<Self::Output>;
}
```

async 块生成的 Future 是 `!Unpin` 的，所以：

```rust
// ❌ 不能这样
async fn foo() {}
let mut fut = foo();
fut.poll(...); // 编译错误！需要 Pin

// ✅ 要这样
let mut fut = Box::pin(foo());
fut.as_mut().poll(...); // OK
```

这就是为什么我们用 `tokio::spawn` 或 `.await` 而不是手动 poll——运行时帮我们处理了 Pin。

---

## Unpin 的自动实现

大多数类型自动实现 `Unpin`：

```rust
// 这些都是 Unpin
struct MyStruct {
    a: i32,
    b: String,
    c: Vec<u8>,
}

// 这个不是 Unpin（因为包含 PhantomPinned）
struct NotUnpin {
    data: String,
    _marker: PhantomPinned,
}
```

---

## Pin 投影（Pin Projection）

当你有一个 `Pin<&mut Struct>`，如何安全地访问字段？

```rust
use std::pin::Pin;

struct MyFuture {
    // Unpin 字段
    count: u32,
    // !Unpin 字段
    inner: InnerFuture,
}

impl MyFuture {
    // 对 Unpin 字段：可以直接获取 &mut
    fn count(self: Pin<&mut Self>) -> &mut u32 {
        // 安全：u32 是 Unpin
        unsafe { &mut self.get_unchecked_mut().count }
    }
    
    // 对 !Unpin 字段：返回 Pin<&mut Field>
    fn inner(self: Pin<&mut Self>) -> Pin<&mut InnerFuture> {
        // 安全：我们保持了 Pin 不变式
        unsafe { self.map_unchecked_mut(|s| &mut s.inner) }
    }
}
```

> 💡 实际项目中可以使用 `pin-project` crate 来安全地处理 Pin 投影。

---

## 总结

| 概念 | 说明 |
|------|------|
| **Pin<P>** | 包装指针，承诺不移动被指向的值 |
| **Unpin** | 标记 trait，表示类型可以安全移动 |
| **!Unpin** | 类型不能被移动（自引用类型） |
| **PhantomPinned** | 让你的类型变成 !Unpin |
| **Box::pin()** | 在堆上创建 Pin |
| **Pin::new()** | 仅对 Unpin 类型创建 Pin |

### 关键记忆点

1. **大多数类型都是 Unpin**，Pin 对它们没有实际约束
2. **async 生成的 Future 是 !Unpin**，这就是 Pin 存在的主要原因
3. 只有在写底层异步代码或自引用结构时才需要直接操作 Pin
4. 日常使用 `.await` 和 `tokio::spawn`，Pin 的细节被隐藏了

---

## 对比其他语言

| 语言 | 自引用问题的处理 |
|------|------------------|
| **PHP/JS** | GC 管理，对象地址对用户透明 |
| **C/C++** | 程序员自己负责，容易出错 |
| **Rust** | 编译期通过 Pin 保证安全 |

---

*下节预告：ManuallyDrop — 手动控制析构*
