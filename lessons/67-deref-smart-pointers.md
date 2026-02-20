# 第 67 课：Deref 与智能指针的魔法

> 日期：2026-02-21  
> 主题：Deref, DerefMut, Box, 以及自动解引用

---

## 先看一个"神奇"的现象

```rust
fn print_str(s: &str) {
    println!("{}", s);
}

let s = String::from("hello");
print_str(&s);  // 🤔 为什么 &String 能传给 &str？
```

这就是 **Deref 强制转换 (Deref Coercion)** 的魔法。

---

## Deref trait 是什么？

```rust
// 标准库定义
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
```

简单说：**Deref 让你定义"解引用"的行为**。

当编译器看到 `*x` 时，实际调用的是 `*(x.deref())`。

---

## String 的 Deref 实现

```rust
// 标准库里是这样的
impl Deref for String {
    type Target = str;
    
    fn deref(&self) -> &str {
        // 返回内部的 str 切片
        &self[..]
    }
}
```

所以：
- `*String` → `str`
- `&String` 可以自动变成 `&str`

---

## Deref Coercion（强制解引用）

编译器会**自动**在需要的地方插入 `.deref()` 调用：

```rust
let s = String::from("hello");

// 你写的
print_str(&s);

// 编译器理解成
print_str(s.deref());  // &String → &str
```

### 多层自动解引用

```rust
let boxed: Box<String> = Box::new(String::from("hi"));

// Box<String> → String → str
print_str(&boxed);  // 自动解两层！
```

规则：编译器会一直解到类型匹配为止。

---

## Box<T> — 最简单的智能指针

```rust
// 在堆上分配
let x: Box<i32> = Box::new(42);

// 使用时，Deref 让它像普通引用一样
println!("{}", *x);  // 42

// 更常用：自动 Deref
fn print_num(n: &i32) {
    println!("{}", n);
}
print_num(&x);  // &Box<i32> → &i32
```

### Box 的用途

```rust
// 1. 递归类型（大小不确定）
enum List {
    Cons(i32, Box<List>),
    Nil,
}

// 2. 大数据避免栈拷贝
let huge = Box::new([0u8; 1_000_000]);

// 3. trait 对象
let animal: Box<dyn Animal> = Box::new(Dog);
```

---

## DerefMut — 可变解引用

```rust
pub trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

```rust
let mut s = String::from("hello");

// &mut String → &mut str
fn modify(s: &mut str) {
    // 注意：str 本身不能改大小
    // 但可以改内容
    s.make_ascii_uppercase();
}

modify(&mut s);
println!("{}", s);  // HELLO
```

---

## 自己实现一个智能指针

```rust
use std::ops::{Deref, DerefMut};

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;
    
    fn deref(&self) -> &T {
        &self.0
    }
}

impl<T> DerefMut for MyBox<T> {
    fn deref_mut(&mut self) -> &mut T {
        &mut self.0
    }
}

// 使用
let x = MyBox::new(5);
assert_eq!(*x, 5);
```

---

## Deref Coercion 规则

| 场景 | 转换 |
|------|------|
| `&T` → `&U` | 当 `T: Deref<Target=U>` |
| `&mut T` → `&mut U` | 当 `T: DerefMut<Target=U>` |
| `&mut T` → `&U` | 当 `T: Deref<Target=U>`（可变变不可变 ✅） |

**注意**：`&T` 不能变成 `&mut U`（不可变不能变可变 ❌）

---

## 常见的 Deref 链

```
Box<T>    → T
Vec<T>    → [T]
String    → str
Rc<T>     → T
Arc<T>    → T
MutexGuard<T> → T
```

---

## 实战技巧

### 1. 用 &self 方法时不用显式解引用

```rust
let s = Box::new(String::from("hello"));

// 不需要 (*s).len()
println!("{}", s.len());  // 自动 Deref
```

### 2. 方法查找会自动 Deref

```rust
let v: Box<Vec<i32>> = Box::new(vec![1, 2, 3]);

// Box<Vec<i32>> → Vec<i32> → [i32]
// 然后找到 first() 方法
v.first();
```

### 3. 函数参数自动转换

```rust
fn process(data: &[u8]) { ... }

let vec = Vec::from([1, 2, 3]);
let boxed = Box::new(vec);

process(&boxed);  // Box<Vec<u8>> → &[u8]
```

---

## 对比 PHP

```php
// PHP 没有 Deref 概念
// 但有类似的 __toString() 魔术方法
class User {
    public function __toString(): string {
        return $this->name;
    }
}

echo $user;  // 自动调用 __toString()
```

Rust 的 Deref 更强大：
- 编译期检查
- 可以多层嵌套
- 不只是 toString，是任意类型

---

## 要点总结

1. **Deref 定义了 `*` 解引用的行为**
2. **Deref Coercion 让类型自动转换**（编译期，零成本）
3. **`&String → &str` 就是 Deref 的功劳**
4. **Box 是最基础的智能指针**
5. **方法调用会自动 Deref 查找**

---

## 下节预告

**Rc 与 Arc — 引用计数智能指针** 🔢

当一个值需要多个所有者时怎么办？

---

*课程笔记：性奴001*
