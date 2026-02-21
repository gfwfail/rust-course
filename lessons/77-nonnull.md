# 第 77 课：NonNull — 非空裸指针的安全封装

> 日期：2026-02-22  
> 主题：std::ptr::NonNull 的原理与使用

---

## 为什么需要 NonNull？

Rust 的裸指针 `*mut T` 和 `*const T` 可以是 null：

```rust
let ptr: *const i32 = std::ptr::null();  // 合法，但危险

// 解引用空指针是 UB
unsafe { *ptr };  // 💥 未定义行为！
```

但在很多数据结构中，我们 **知道** 指针永远不会是 null：
- `Box<T>` 内部的指针
- `Vec<T>` 分配的内存
- 大多数堆分配的数据

`NonNull<T>` 就是对此的类型级保证。

---

## NonNull 是什么？

```rust
use std::ptr::NonNull;

// NonNull 是一个保证非空的指针
// 它的定义大致是：
// struct NonNull<T> {
//     pointer: *const T,  // 内部就是裸指针
// }
```

**核心特性**：
- 保证永远不是 null
- 协变（covariant），和 `&T` 一样
- 可以利用 niche 优化

---

## 基本用法

```rust
use std::ptr::NonNull;

let mut x = 42i32;

// 从可变引用创建（最安全）
let ptr: NonNull<i32> = NonNull::from(&mut x);

// 从裸指针创建（需要你保证非空）
let raw_ptr: *mut i32 = &mut x;
let ptr: NonNull<i32> = NonNull::new(raw_ptr).expect("指针为空！");

// 或者 unsafe 版本
let ptr: NonNull<i32> = unsafe { NonNull::new_unchecked(raw_ptr) };

// 使用
unsafe {
    *ptr.as_ptr() = 100;  // 修改值
    println!("{}", *ptr.as_ptr());  // 100
}
```

---

## 对比 Option<NonNull> vs 原始指针

```rust
use std::ptr::NonNull;
use std::mem::size_of;

// 裸指针可以是 null
let raw: *mut i32 = std::ptr::null_mut();

// NonNull 保证非空，所以 Option<NonNull<T>> 
// 利用 null 作为 None 的表示
println!("{}", size_of::<*mut i32>());              // 8
println!("{}", size_of::<NonNull<i32>>());          // 8
println!("{}", size_of::<Option<NonNull<i32>>>()); // 8 ！（niche 优化）

// 对比：Option<*mut T> 需要额外空间
println!("{}", size_of::<Option<*mut i32>>());     // 16
```

**这就是 niche 优化**：因为 NonNull 永远不是 null，Option 可以用 null 值表示 None。

---

## 常用方法

```rust
use std::ptr::NonNull;

let mut value = String::from("hello");
let ptr = NonNull::from(&mut value);

// as_ptr() - 获取裸指针
let raw: *mut String = ptr.as_ptr();

// as_ref() - 获取不可变引用（unsafe）
let r: &String = unsafe { ptr.as_ref() };
println!("{}", r);  // "hello"

// as_mut() - 获取可变引用（unsafe）
let r: &mut String = unsafe { ptr.as_mut() };
r.push_str(" world");
println!("{}", r);  // "hello world"

// cast() - 类型转换
let byte_ptr: NonNull<u8> = ptr.cast::<u8>();
```

---

## 实际应用：实现链表节点

```rust
use std::ptr::NonNull;

struct Node<T> {
    value: T,
    // 用 Option<NonNull> 而不是 Option<Box>
    // 这样可以有多个指针指向同一个节点
    next: Option<NonNull<Node<T>>>,
    prev: Option<NonNull<Node<T>>>,
}

struct LinkedList<T> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
}

impl<T> LinkedList<T> {
    fn new() -> Self {
        LinkedList {
            head: None,
            tail: None,
            len: 0,
        }
    }
    
    fn push_back(&mut self, value: T) {
        // 在堆上分配新节点
        let node = Box::new(Node {
            value,
            next: None,
            prev: self.tail,
        });
        
        // 转换为 NonNull
        let node_ptr = unsafe { 
            NonNull::new_unchecked(Box::into_raw(node)) 
        };
        
        // 更新链接
        match self.tail {
            Some(mut tail) => {
                unsafe { tail.as_mut().next = Some(node_ptr) };
            }
            None => {
                self.head = Some(node_ptr);
            }
        }
        
        self.tail = Some(node_ptr);
        self.len += 1;
    }
}
```

---

## NonNull vs Box vs &mut

| 类型 | 所有权 | 可空 | 用途 |
|------|--------|------|------|
| `Box<T>` | 独占所有权 | 不可空 | 堆分配的独占数据 |
| `&mut T` | 借用 | 不可空 | 临时可变访问 |
| `NonNull<T>` | 无所有权语义 | 不可空 | 底层数据结构、FFI |
| `*mut T` | 无所有权语义 | 可空 | 最原始的指针 |

**什么时候用 NonNull？**
- 实现自定义数据结构（链表、树等）
- 需要多个可变"指针"指向同一数据
- 与 C 代码交互，但你知道指针不是 null

---

## 与 PHP/JS 对比

```php
// PHP - 引用随便用，null 检查是运行时的
$node = new Node();
$node->next = null;  // 合法
$node->next->value;  // 运行时错误

// 没有类型级别的"这个引用永远不是 null"
```

```rust
// Rust - 类型告诉你一切
let node: NonNull<Node>;  // 编译器知道这不会是 null
let maybe_node: Option<NonNull<Node>>;  // 可能为空
```

---

## 创建 NonNull 的几种方式

```rust
use std::ptr::NonNull;

let mut x = 42;

// 1. 从引用（最安全）
let ptr = NonNull::from(&mut x);

// 2. 从裸指针，返回 Option
let ptr = NonNull::new(&mut x as *mut i32);  // Some(...)
let ptr = NonNull::new(std::ptr::null_mut::<i32>());  // None

// 3. 从裸指针，unsafe（你保证非空）
let ptr = unsafe { NonNull::new_unchecked(&mut x) };

// 4. dangling() - 创建一个悬垂但非空的指针
// 用于 ZST（零大小类型）或占位
let ptr: NonNull<i32> = NonNull::dangling();
// ⚠️ 这个指针不能解引用！只是为了满足"非空"的要求
```

---

## dangling() 的用途

```rust
use std::ptr::NonNull;

struct MyVec<T> {
    ptr: NonNull<T>,
    len: usize,
    cap: usize,
}

impl<T> MyVec<T> {
    fn new() -> Self {
        MyVec {
            // 空 Vec 用 dangling 指针
            // 因为没分配内存，但 NonNull 要求非空
            ptr: NonNull::dangling(),
            len: 0,
            cap: 0,
        }
    }
}

// 标准库的 Vec 就是这样实现的！
```

---

## 总结

| 概念 | 说明 |
|------|------|
| `NonNull<T>` | 保证非空的裸指针封装 |
| `NonNull::new(ptr)` | 从裸指针创建，返回 Option |
| `NonNull::new_unchecked(ptr)` | unsafe，你保证非空 |
| `NonNull::from(&mut x)` | 从引用创建（最安全） |
| `NonNull::dangling()` | 创建悬垂指针（不可解引用） |
| `.as_ptr()` | 获取裸指针 |
| `.as_ref()` / `.as_mut()` | unsafe，获取引用 |
| `Option<NonNull<T>>` | 大小 = 一个指针（niche 优化） |

**使用场景**：
- 实现自定义集合类型
- 底层内存管理
- FFI 交互
- 需要"非空指针"类型保证的地方

---

*下节课：Unique — 独占指针（标准库内部类型）*
