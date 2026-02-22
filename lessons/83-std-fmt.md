# 第 83 课：std::fmt — 格式化输出的艺术

> 日期：2026-02-23  
> 主题：格式化输出、Display、Debug、Formatter

---

## 📍 为什么要学这个？

每次你写 `println!("{}", x)` 或 `format!("{:?}", x)`，背后都是 `std::fmt` 在工作。理解它能让你：
- 自定义类型的打印格式
- 写出更优雅的调试输出
- 理解 `Display` vs `Debug` 的本质区别

---

## 🎯 核心概念：两大 Trait

```rust
// Display — 给用户看的
trait Display {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}

// Debug — 给程序员调试用的
trait Debug {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}
```

**类比 PHP/Laravel：**
- `Display` ≈ `__toString()` — 用户友好的输出
- `Debug` ≈ `dd()` 或 `var_dump()` — 调试用的详细输出

---

## 🔧 实战：为自定义类型实现格式化

```rust
use std::fmt;

struct Money {
    amount: i64,    // 分
    currency: &'static str,
}

// Display: 用户看到的格式
impl fmt::Display for Money {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let dollars = self.amount as f64 / 100.0;
        write!(f, "{}{:.2}", self.currency, dollars)
    }
}

// Debug: 调试用的格式
impl fmt::Debug for Money {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Money")
            .field("amount", &self.amount)
            .field("currency", &self.currency)
            .finish()
    }
}

fn main() {
    let price = Money { amount: 9999, currency: "$" };
    
    println!("{}", price);   // $99.99
    println!("{:?}", price); // Money { amount: 9999, currency: "$" }
}
```

---

## 🎨 格式化占位符详解

```rust
let x = 42;
let pi = 3.14159;
let name = "Rust";

// 基础占位符
println!("{}", x);      // 42 (Display)
println!("{:?}", x);    // 42 (Debug)
println!("{:#?}", x);   // 42 (Pretty Debug，多行缩进)

// 数字格式化
println!("{:05}", x);   // 00042 (补零到5位)
println!("{:>8}", x);   // "      42" (右对齐，宽度8)
println!("{:<8}", x);   // "42      " (左对齐)
println!("{:^8}", x);   // "   42   " (居中)

// 进制转换
println!("{:b}", x);    // 101010 (二进制)
println!("{:o}", x);    // 52 (八进制)
println!("{:x}", x);    // 2a (十六进制小写)
println!("{:X}", x);    // 2A (十六进制大写)
println!("{:#x}", x);   // 0x2a (带前缀)

// 浮点数
println!("{:.2}", pi);  // 3.14 (2位小数)
println!("{:8.2}", pi); // "    3.14" (宽度8，2位小数)
println!("{:e}", pi);   // 3.14159e0 (科学计数法)
```

---

## 🛠️ Formatter 的高级用法

```rust
use std::fmt;

struct Color {
    r: u8, g: u8, b: u8,
}

impl fmt::Display for Color {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // 检查用户请求的格式
        if f.alternate() {
            // {:#} 格式：输出 #RRGGBB
            write!(f, "#{:02X}{:02X}{:02X}", self.r, self.g, self.b)
        } else {
            // {} 格式：输出 rgb(r, g, b)
            write!(f, "rgb({}, {}, {})", self.r, self.g, self.b)
        }
    }
}

impl fmt::LowerHex for Color {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // {:x} 格式
        write!(f, "{:02x}{:02x}{:02x}", self.r, self.g, self.b)
    }
}

fn main() {
    let red = Color { r: 255, g: 0, b: 0 };
    
    println!("{}", red);    // rgb(255, 0, 0)
    println!("{:#}", red);  // #FF0000
    println!("{:x}", red);  // ff0000
}
```

---

## 📋 常用格式化 Trait 一览

| Trait | 占位符 | 用途 |
|-------|--------|------|
| `Display` | `{}` | 用户友好输出 |
| `Debug` | `{:?}` | 调试输出 |
| `Binary` | `{:b}` | 二进制 |
| `Octal` | `{:o}` | 八进制 |
| `LowerHex` | `{:x}` | 十六进制小写 |
| `UpperHex` | `{:X}` | 十六进制大写 |
| `LowerExp` | `{:e}` | 科学计数法小写 |
| `UpperExp` | `{:E}` | 科学计数法大写 |
| `Pointer` | `{:p}` | 指针地址 |

---

## 💡 命名参数和位置参数

```rust
// 位置参数
println!("{0} + {1} = {2}", 1, 2, 3);  // 1 + 2 = 3
println!("{0} {0} {1}", "ho", "ha");   // ho ho ha

// 命名参数
println!("{name} is {age} years old", 
    name = "Ferris", 
    age = 5
);  // Ferris is 5 years old

// 混合使用
println!("{} {} {name}", 1, 2, name = "three");
// 1 2 three
```

---

## 🚀 Debug 的自动派生

```rust
// 大多数时候，直接 derive 就够了
#[derive(Debug)]
struct User {
    id: u64,
    name: String,
    email: String,
}

fn main() {
    let user = User {
        id: 1,
        name: "Alice".into(),
        email: "alice@example.com".into(),
    };
    
    // {:?} 单行
    println!("{:?}", user);
    // User { id: 1, name: "Alice", email: "alice@example.com" }
    
    // {:#?} 多行美化
    println!("{:#?}", user);
    // User {
    //     id: 1,
    //     name: "Alice",
    //     email: "alice@example.com",
    // }
}
```

---

## 🔍 write! vs println!

```rust
use std::fmt::Write;  // 注意：需要引入 Write trait

let mut buffer = String::new();

// write! 写入任何实现了 Write trait 的地方
write!(buffer, "Hello, {}!", "world").unwrap();

// println! 写入 stdout
println!("{}", buffer);  // Hello, world!

// format! 返回 String
let s = format!("Value: {}", 42);
```

---

## 📝 课后思考

1. **为什么 `Display` 不能 derive，但 `Debug` 可以？**
   - `Display` 是给最终用户看的，需要人工设计输出格式
   - `Debug` 是给程序员看的，结构化输出可以自动生成

2. **`write!` 和 `println!` 有什么区别？**
   - `write!` 写入任何实现了 `Write` 的地方（String、文件等），返回 `Result`
   - `println!` 专门写入 stdout，panic on error

---

## 💡 小技巧

```rust
// 用 {:?} 快速打印 Vec
let nums = vec![1, 2, 3];
println!("{:?}", nums);  // [1, 2, 3]

// 用 {:#?} 美化嵌套结构
let nested = vec![vec![1, 2], vec![3, 4]];
println!("{:#?}", nested);
// [
//     [1, 2],
//     [3, 4],
// ]
```

---

*下节课继续探索标准库！*
