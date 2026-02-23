# 第 87 课：std::clone — 克隆与复制的艺术

> 日期：2026-02-23  
> 主题：Clone 和 Copy trait

---

## 📌 核心问题：为什么需要 Clone？

在 Rust 中，赋值默认是 **move**（移动）：

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 被 move 了！
// println!("{}", s1);  // ❌ 编译错误：s1 已无效
```

但有时候我们真的需要一份独立的副本，这就是 `Clone` 的用武之地：

```rust
let s1 = String::from("hello");
let s2 = s1.clone();  // 显式克隆
println!("{} {}", s1, s2);  // ✅ 都能用
```

---

## 🔧 Clone trait

```rust
pub trait Clone: Sized {
    fn clone(&self) -> Self;
    
    fn clone_from(&mut self, source: &Self) {
        *self = source.clone();  // 默认实现
    }
}
```

### 自动派生

大多数情况用 `#[derive(Clone)]` 就够了：

```rust
#[derive(Clone)]
struct User {
    name: String,
    age: u32,
}

let user1 = User {
    name: "张三".into(),
    age: 25,
};
let user2 = user1.clone();  // 深拷贝
```

### 手动实现

有时需要自定义克隆逻辑：

```rust
struct Connection {
    id: u64,
    pool: Arc<ConnectionPool>,
}

impl Clone for Connection {
    fn clone(&self) -> Self {
        // 生成新 ID，但共享连接池
        Self {
            id: generate_new_id(),
            pool: Arc::clone(&self.pool),
        }
    }
}
```

---

## ⚡ Copy trait — 隐式克隆

```rust
pub trait Copy: Clone { }
```

`Copy` 是一个 **marker trait**（标记特征），没有方法。它告诉编译器："这个类型可以按位复制，不需要特殊处理"。

### Copy 的魔力

实现了 `Copy` 的类型，赋值时自动复制而不是 move：

```rust
let x: i32 = 5;
let y = x;  // x 被复制，不是移动
println!("{} {}", x, y);  // ✅ 都能用
```

### 哪些类型实现了 Copy？

1. **所有整数类型**: `i8, i16, i32, i64, i128, isize, u8, u16...`
2. **浮点类型**: `f32, f64`
3. **布尔**: `bool`
4. **字符**: `char`
5. **元组** (如果所有元素都是 Copy): `(i32, bool)`
6. **数组** (如果元素是 Copy): `[i32; 5]`
7. **共享引用**: `&T`
8. **裸指针**: `*const T`, `*mut T`
9. **函数指针**: `fn(i32) -> i32`

### 为什么 String 不是 Copy？

因为 `String` 在堆上有数据。如果按位复制，两个 `String` 会指向同一块堆内存，drop 时会 double free！

```rust
// 假设 String 能 Copy（实际不能）
let s1 = String::from("hello");
let s2 = s1;  // 如果是 Copy，两者指向同一内存
// s1 和 s2 各自 drop 时，同一块内存被释放两次！💥
```

---

## 🆚 Clone vs Copy

| 特性 | Clone | Copy |
|------|-------|------|
| 调用方式 | 显式 `.clone()` | 隐式自动 |
| 性能开销 | 可能很大（深拷贝） | 很小（按位复制） |
| 语义 | 可能有副作用 | 纯粹的值复制 |
| 适用类型 | 几乎所有类型 | 只有"简单"类型 |

---

## 💡 实战技巧

### 1. clone_from 优化

当你要覆盖一个已有的值时，`clone_from` 可能更高效：

```rust
let mut buffer = String::with_capacity(1024);

// 每次都分配新内存
buffer = source.clone();

// 可能复用已有容量
buffer.clone_from(&source);
```

### 2. Clone on Write

之前讲过的 `Cow<T>` 就是利用 Clone 实现延迟复制：

```rust
use std::borrow::Cow;

fn process(input: Cow<str>) {
    // 只有真正需要修改时才 clone
}
```

### 3. 派生 Copy 的条件

要派生 `Copy`，所有字段必须是 `Copy`：

```rust
#[derive(Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}  // ✅ f64 是 Copy

#[derive(Clone, Copy)]
struct User {
    name: String,  // ❌ String 不是 Copy
}  // 编译错误！
```

### 4. Rc/Arc 的克隆

智能指针的 clone 只增加引用计数，不复制底层数据：

```rust
use std::rc::Rc;

let a = Rc::new(vec![1, 2, 3]);
let b = Rc::clone(&a);  // 只是增加计数，超级便宜！

// 惯例：用 Rc::clone(&x) 而非 x.clone()
// 明确表示这是"浅"克隆
```

---

## 🎓 PHP/Laravel 对比

```php
// PHP: 对象默认是引用赋值
$user1 = new User();
$user2 = $user1;  // 同一个对象！

// 要真正复制
$user2 = clone $user1;  // 调用 __clone()

// Rust 的 Clone 类似 PHP 的 clone 关键字
// Rust 的 Copy 没有直接对应物
```

---

## 📝 小测验

```rust
// 问题：以下哪些类型可以实现 Copy？

struct A(i32, i32);           // ✅ 全是 Copy 类型
struct B(String);             // ❌ String 不是 Copy
struct C<'a>(&'a str);        // ✅ 引用是 Copy
struct D(Box<i32>);           // ❌ Box 不是 Copy
struct E;                     // ✅ 空结构体是 Copy
```

---

## 🔑 总结

| 要点 | 说明 |
|------|------|
| `Clone` | 显式深拷贝，可能有开销 |
| `Copy` | 隐式按位复制，零开销 |
| Copy ⊂ Clone | Copy 必须同时实现 Clone |
| 堆数据不能 Copy | String, Vec, Box 都不行 |
| Rc::clone | 惯用写法，表明是浅克隆 |

---

**下节预告**: `std::marker` — Send, Sync, Sized, Unpin 这些神秘的 marker traits 🏷️
