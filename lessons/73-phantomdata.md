# 第 73 课：PhantomData — 幽灵数据与零大小类型

> 日期：2026-02-21  
> 主题：零大小类型与类型系统的高级技巧

---

## 📍 什么是 PhantomData？

```rust
use std::marker::PhantomData;

struct MyType<T> {
    data: u64,
    _marker: PhantomData<T>,  // 不占内存！
}
```

**PhantomData = 幽灵数据**，在类型系统中"存在"，但运行时**零大小、零开销**。

---

## 🤔 为什么需要它？

### 问题：未使用的泛型参数

```rust
// ❌ 编译错误！
struct MyType<T> {
    id: u64,
    // T 没用到，编译器报错：
    // error: parameter `T` is never used
}
```

Rust 编译器要求：**泛型参数必须被使用**。

### 解决方案：PhantomData

```rust
use std::marker::PhantomData;

// ✅ 编译通过
struct MyType<T> {
    id: u64,
    _marker: PhantomData<T>,  // "假装"用了 T
}
```

---

## 💡 经典场景 1：类型标记（Type Tag）

用不同类型区分同一结构体的不同"身份"：

```rust
use std::marker::PhantomData;

// 状态标记（零大小类型）
struct Draft;
struct Published;
struct Archived;

// 文章结构体
struct Article<State> {
    title: String,
    content: String,
    _state: PhantomData<State>,
}

impl Article<Draft> {
    fn new(title: String) -> Self {
        Article {
            title,
            content: String::new(),
            _state: PhantomData,
        }
    }
    
    fn publish(self) -> Article<Published> {
        Article {
            title: self.title,
            content: self.content,
            _state: PhantomData,
        }
    }
}

impl Article<Published> {
    fn archive(self) -> Article<Archived> {
        Article {
            title: self.title,
            content: self.content,
            _state: PhantomData,
        }
    }
}

// 只有 Published 的文章能被阅读
impl Article<Published> {
    fn read(&self) -> &str {
        &self.content
    }
}
```

用 Laravel 类比：
```php
// PHP 只能运行时检查状态
$article->publish();
if ($article->status !== 'published') {
    throw new Exception("Can't read unpublished article");
}
```

**Rust 用 PhantomData 实现编译期状态机！**

---

## 🔥 场景 2：类型安全的 ID

```rust
use std::marker::PhantomData;

// 不同表的 ID，类型不同，值相同
struct Id<T> {
    value: u64,
    _marker: PhantomData<T>,
}

struct User;
struct Order;

impl<T> Id<T> {
    fn new(value: u64) -> Self {
        Id { value, _marker: PhantomData }
    }
}

fn get_user(id: Id<User>) {
    println!("Getting user {}", id.value);
}

fn get_order(id: Id<Order>) {
    println!("Getting order {}", id.value);
}

fn main() {
    let user_id: Id<User> = Id::new(42);
    let order_id: Id<Order> = Id::new(42);
    
    get_user(user_id);    // ✅
    // get_user(order_id); // ❌ 编译错误！类型不匹配
}
```

用 Laravel 类比：
```php
// PHP 中只有 int，容易混淆
function getUser(int $id) { }
function getOrder(int $id) { }

$userId = 42;
$orderId = 42;
getUser($orderId);  // 不小心传错了，但编译通过 😱
```

**Rust 的 PhantomData 让 ID 类型安全！**

---

## 🎯 场景 3：生命周期标记

```rust
use std::marker::PhantomData;

// 迭代器持有引用的生命周期
struct Iter<'a, T> {
    ptr: *const T,           // 裸指针，没有生命周期信息
    end: *const T,
    _marker: PhantomData<&'a T>,  // 告诉编译器：我和 'a 相关
}

impl<'a, T> Iter<'a, T> {
    fn new(slice: &'a [T]) -> Self {
        let ptr = slice.as_ptr();
        let end = unsafe { ptr.add(slice.len()) };
        Iter {
            ptr,
            end,
            _marker: PhantomData,
        }
    }
}
```

**为什么需要这个？**

裸指针 `*const T` 没有生命周期信息，编译器不知道这个迭代器依赖于某个切片。`PhantomData<&'a T>` 告诉编译器："我持有一个 `&'a T` 的引用语义"。

---

## ⚡ 场景 4：协变与逆变标记

这个更高级，涉及到 Rust 的 variance（变型）：

```rust
use std::marker::PhantomData;

// PhantomData<T> 让结构体对 T 协变（covariant）
struct Covariant<T> {
    _marker: PhantomData<T>,
}

// PhantomData<fn(T)> 让结构体对 T 逆变（contravariant）
struct Contravariant<T> {
    _marker: PhantomData<fn(T)>,
}

// PhantomData<fn(T) -> T> 让结构体对 T 不变（invariant）
struct Invariant<T> {
    _marker: PhantomData<fn(T) -> T>,
}
```

什么意思？

- **协变**：`PhantomData<T>` — 如果 `T: 'static`，则 `Covariant<T>` 也能用于需要更长生命周期的地方
- **逆变**：`PhantomData<fn(T)>` — 相反
- **不变**：`PhantomData<fn(T) -> T>` — T 的生命周期必须严格匹配

---

## 📦 PhantomData 的内存布局

```rust
use std::marker::PhantomData;
use std::mem::size_of;

struct WithPhantom<T> {
    id: u64,
    _marker: PhantomData<T>,
}

fn main() {
    println!("size of u64: {}", size_of::<u64>());  // 8
    println!("size of PhantomData<u64>: {}", size_of::<PhantomData<u64>>());  // 0
    println!("size of WithPhantom<u64>: {}", size_of::<WithPhantom<u64>>());  // 8（不是 16！）
}
```

**PhantomData 是零大小类型（ZST），完全不占内存！**

---

## 🎭 实战：类型安全的构建器模式

```rust
use std::marker::PhantomData;

// 构建器状态
struct NoName;
struct HasName;
struct NoEmail;
struct HasEmail;

struct UserBuilder<N, E> {
    name: Option<String>,
    email: Option<String>,
    _name_state: PhantomData<N>,
    _email_state: PhantomData<E>,
}

impl UserBuilder<NoName, NoEmail> {
    fn new() -> Self {
        UserBuilder {
            name: None,
            email: None,
            _name_state: PhantomData,
            _email_state: PhantomData,
        }
    }
}

impl<E> UserBuilder<NoName, E> {
    fn name(self, name: &str) -> UserBuilder<HasName, E> {
        UserBuilder {
            name: Some(name.to_string()),
            email: self.email,
            _name_state: PhantomData,
            _email_state: PhantomData,
        }
    }
}

impl<N> UserBuilder<N, NoEmail> {
    fn email(self, email: &str) -> UserBuilder<N, HasEmail> {
        UserBuilder {
            name: self.name,
            email: Some(email.to_string()),
            _name_state: PhantomData,
            _email_state: PhantomData,
        }
    }
}

// 只有当 name 和 email 都设置了，才能 build
impl UserBuilder<HasName, HasEmail> {
    fn build(self) -> User {
        User {
            name: self.name.unwrap(),
            email: self.email.unwrap(),
        }
    }
}

struct User {
    name: String,
    email: String,
}

fn main() {
    let user = UserBuilder::new()
        .name("Alice")
        .email("alice@example.com")
        .build();  // ✅ 编译通过
    
    // let incomplete = UserBuilder::new()
    //     .name("Bob")
    //     .build();  // ❌ 编译错误！没有 email
}
```

**这叫 Typestate Pattern（类型状态模式）！**

---

## ⚠️ 常见陷阱

### 陷阱 1：PhantomData 影响 Drop 检查

```rust
use std::marker::PhantomData;

struct MyBox<T> {
    ptr: *mut T,
    _marker: PhantomData<T>,  // 影响 drop check！
}
```

`PhantomData<T>` 告诉编译器："我拥有 T"。这会影响借用检查器。

如果你只是"借用"T，应该用：
```rust
_marker: PhantomData<&'a T>  // 我借用 T
```

如果你不想影响 drop check：
```rust
_marker: PhantomData<*const T>  // 原始指针，不影响 drop
```

### PhantomData 变体总结

| 形式 | 含义 | Variance |
|------|------|----------|
| `PhantomData<T>` | 拥有 T | 协变 |
| `PhantomData<&'a T>` | 借用 T | 协变 |
| `PhantomData<*const T>` | 不影响 drop | 协变 |
| `PhantomData<fn(T)>` | 逆变标记 | 逆变 |
| `PhantomData<fn(T) -> T>` | 不变标记 | 不变 |

---

## 🧠 本课要点

1. **PhantomData 是零大小类型**，运行时零开销
2. 主要用途：
   - 让编译器接受未使用的泛型参数
   - 类型标记/状态机（Typestate Pattern）
   - 生命周期标记（告诉编译器引用关系）
   - 类型安全的 ID
3. `PhantomData<T>` 表示"拥有 T"
4. `PhantomData<&'a T>` 表示"借用 T"
5. 这是 Rust 类型系统的高级技巧，在底层库中常见

---

## 📝 练习思考

1. 为什么 Rust 不允许未使用的泛型参数？
   - 答案：编译器需要知道泛型参数如何影响类型的行为（variance、drop 等）

2. PhantomData 为什么叫"幽灵"？
   - 答案：因为它在类型系统中存在，但运行时不占任何内存

3. Typestate Pattern 有什么好处？
   - 答案：把运行时检查变成编译期检查，非法状态转换直接编译失败

---

## 📚 相关文档

- [std::marker::PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html)
- [The Rustonomicon - PhantomData](https://doc.rust-lang.org/nomicon/phantom-data.html)
- [Variance in Rust](https://doc.rust-lang.org/nomicon/subtyping.html)

---

*下节课预告：OnceCell 与 OnceLock — 一次性初始化*
