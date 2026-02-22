# 第 80 课：std::ops — 操作符重载

> 日期：2026-02-22  
> 主题：让你的类型支持 +、-、*、[]、() 等操作符

---

## 概述

`std::ops` 模块定义了一系列 trait，让你可以为自定义类型实现各种操作符。这是 Rust 类型系统的强大特性之一。

---

## 为什么需要操作符重载？

假设你写了一个 `Point` 结构体：

```rust
struct Point {
    x: i32,
    y: i32,
}

let a = Point { x: 1, y: 2 };
let b = Point { x: 3, y: 4 };

// 这样写多爽？
let c = a + b;  // ❌ 编译错误！
```

Rust 不知道两个 `Point` 怎么"加"。你得告诉它。

---

## 📐 算术运算符

### Add — 加法 `+`

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;  // 加法结果的类型
    
    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    let a = Point { x: 1, y: 2 };
    let b = Point { x: 3, y: 4 };
    let c = a + b;  // ✅ 现在可以了！
    
    println!("{:?}", c);  // Point { x: 4, y: 6 }
}
```

**trait 定义长这样：**

```rust
pub trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

`Rhs = Self` 表示默认右操作数和自己同类型。

---

### 不同类型相加

```rust
use std::ops::Add;

#[derive(Debug)]
struct Meters(f64);

#[derive(Debug)]
struct Centimeters(f64);

// Meters + Centimeters = Meters
impl Add<Centimeters> for Meters {
    type Output = Meters;
    
    fn add(self, other: Centimeters) -> Meters {
        Meters(self.0 + other.0 / 100.0)
    }
}

fn main() {
    let m = Meters(2.0);
    let cm = Centimeters(50.0);
    
    let result = m + cm;
    println!("{:?}", result);  // Meters(2.5)
}
```

---

### 完整算术运算符列表

| Trait | 操作符 | 方法 |
|-------|--------|------|
| `Add` | `+` | `add(self, rhs)` |
| `Sub` | `-` | `sub(self, rhs)` |
| `Mul` | `*` | `mul(self, rhs)` |
| `Div` | `/` | `div(self, rhs)` |
| `Rem` | `%` | `rem(self, rhs)` |
| `Neg` | `-x` | `neg(self)` |

```rust
use std::ops::{Sub, Mul, Neg};

impl Sub for Point {
    type Output = Point;
    fn sub(self, other: Point) -> Point {
        Point { x: self.x - other.x, y: self.y - other.y }
    }
}

impl Mul<i32> for Point {
    type Output = Point;
    fn mul(self, scalar: i32) -> Point {
        Point { x: self.x * scalar, y: self.y * scalar }
    }
}

impl Neg for Point {
    type Output = Point;
    fn neg(self) -> Point {
        Point { x: -self.x, y: -self.y }
    }
}

// 现在可以这样用
let a = Point { x: 5, y: 10 };
let b = a * 3;       // Point { x: 15, y: 30 }
let c = -a;          // Point { x: -5, y: -10 }
```

---

## 📝 赋值运算符 (+=, -= 等)

注意：`a += b` 和 `a = a + b` 是**不同的 trait**！

```rust
use std::ops::AddAssign;

impl AddAssign for Point {
    fn add_assign(&mut self, other: Point) {
        self.x += other.x;
        self.y += other.y;
    }
}

fn main() {
    let mut a = Point { x: 1, y: 2 };
    a += Point { x: 3, y: 4 };
    
    println!("{:?}", a);  // Point { x: 4, y: 6 }
}
```

| Trait | 操作符 |
|-------|--------|
| `AddAssign` | `+=` |
| `SubAssign` | `-=` |
| `MulAssign` | `*=` |
| `DivAssign` | `/=` |
| `RemAssign` | `%=` |

---

## 📦 Index 和 IndexMut — 索引操作 `[]`

```rust
use std::ops::{Index, IndexMut};

struct Matrix {
    data: Vec<Vec<i32>>,
}

// 只读索引 matrix[row]
impl Index<usize> for Matrix {
    type Output = Vec<i32>;
    
    fn index(&self, row: usize) -> &Vec<i32> {
        &self.data[row]
    }
}

// 可变索引 matrix[row] = ...
impl IndexMut<usize> for Matrix {
    fn index_mut(&mut self, row: usize) -> &mut Vec<i32> {
        &mut self.data[row]
    }
}

fn main() {
    let mut m = Matrix {
        data: vec![
            vec![1, 2, 3],
            vec![4, 5, 6],
        ],
    };
    
    println!("{:?}", m[0]);     // [1, 2, 3]
    println!("{}", m[1][1]);    // 5
    
    m[0][0] = 100;
    println!("{:?}", m[0]);     // [100, 2, 3]
}
```

---

## 📞 Fn, FnMut, FnOnce — 函数调用 `()`

让你的类型可以像函数一样调用！

```rust
struct Adder {
    value: i32,
}

// 目前 Fn traits 不能直接 impl（unstable）
// 但你可以通过闭包捕获来模拟

// 常见做法：impl 一个 call 方法
impl Adder {
    fn call(&self, x: i32) -> i32 {
        self.value + x
    }
}

// 或者返回闭包
fn make_adder(value: i32) -> impl Fn(i32) -> i32 {
    move |x| value + x
}

fn main() {
    let add_5 = make_adder(5);
    println!("{}", add_5(10));  // 15
}
```

**注意：** 直接给自定义类型实现 `Fn` 目前需要 nightly Rust。

---

## 🎭 Deref 和 DerefMut — 解引用 `*`

上节课提过，这里复习一下：

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> Deref for MyBox<T> {
    type Target = T;
    
    fn deref(&self) -> &T {
        &self.0
    }
}

fn main() {
    let x = MyBox(5);
    
    // * 操作符
    assert_eq!(*x, 5);
    
    // 自动解引用
    let s = MyBox(String::from("hello"));
    print_str(&s);  // MyBox<String> 自动转为 &str
}

fn print_str(s: &str) {
    println!("{}", s);
}
```

---

## 🔀 位运算符

```rust
use std::ops::{BitAnd, BitOr, BitXor, Not, Shl, Shr};

#[derive(Debug, Clone, Copy)]
struct Flags(u8);

impl BitAnd for Flags {
    type Output = Flags;
    fn bitand(self, rhs: Flags) -> Flags {
        Flags(self.0 & rhs.0)
    }
}

impl BitOr for Flags {
    type Output = Flags;
    fn bitor(self, rhs: Flags) -> Flags {
        Flags(self.0 | rhs.0)
    }
}

impl Not for Flags {
    type Output = Flags;
    fn not(self) -> Flags {
        Flags(!self.0)
    }
}

fn main() {
    let read = Flags(0b001);
    let write = Flags(0b010);
    let exec = Flags(0b100);
    
    let rw = read | write;
    println!("{:b}", rw.0);  // 11
    
    let has_read = (rw & read).0 != 0;
    println!("has read: {}", has_read);  // true
}
```

位运算符列表：

| Trait | 操作符 |
|-------|--------|
| `BitAnd` | `&` |
| `BitOr` | `\|` |
| `BitXor` | `^` |
| `Not` | `!` |
| `Shl` | `<<` |
| `Shr` | `>>` |

---

## 📊 总结

| 类别 | Traits | 操作符 |
|------|--------|--------|
| 算术 | Add, Sub, Mul, Div, Rem, Neg | + - * / % -x |
| 赋值 | AddAssign, SubAssign... | += -= *= /= %= |
| 索引 | Index, IndexMut | [] |
| 解引用 | Deref, DerefMut | * |
| 位运算 | BitAnd, BitOr, BitXor, Not, Shl, Shr | & \| ^ ! << >> |

---

## 💡 最佳实践

```rust
// ✅ 保持语义一致
// Point + Point = Point (合理)
// String + &str = String (标准库就是这样)

// ✅ 实现相关的 trait 配套
// 如果实现了 Add，通常也该实现 AddAssign
// 如果实现了 Index，考虑是否需要 IndexMut

// ✅ 对于数学类型，考虑实现完整的运算符
// 比如向量、矩阵、复数等

// ❌ 不要滥用操作符重载
// << 用来输出？（C++ iostream 的坑）
// 让操作符保持直觉含义
```

---

## 延伸阅读

- [std::ops 文档](https://doc.rust-lang.org/std/ops/)
- [The Rust Book - Operator Overloading](https://doc.rust-lang.org/book/appendix-02-operators.html)
