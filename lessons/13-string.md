# 第13课：String 深入

Rust 字符串是很多新手最头疼的地方，今天来彻底搞懂。

## 两种字符串，别搞混了

```rust
// &str - 字符串切片，不可变，存在栈上或程序只读段
let s1: &str = "hello";

// String - 可增长的字符串，存在堆上
let s2: String = String::from("hello");
```

| 类型 | 存储位置 | 可变性 | 典型来源 |
|------|---------|--------|---------|
| `&str` | 栈/静态区 | 不可变 | 字面量、切片 |
| `String` | 堆 | 可变 | 动态创建 |

## 相互转换

```rust
// &str → String
let s1 = "hello".to_string();
let s2 = String::from("hello");
let s3 = "hello".to_owned();

// String → &str（自动解引用）
let string = String::from("hello");
let slice: &str = &string;  // 自动 Deref
```

## 拼接字符串

```rust
// 方法1：push / push_str（修改原 String）
let mut s = String::from("Hello");
s.push(' ');           // push 单个字符
s.push_str("World");   // push_str 拼接 &str
println!("{}", s);     // "Hello World"

// 方法2：+ 运算符（消耗第一个 String）
let s1 = String::from("Hello, ");
let s2 = String::from("World!");
let s3 = s1 + &s2;  // 注意：s1 被移动，s2 借用
// println!("{}", s1);  // ❌ s1 已无效

// 方法3：format! 宏（最灵活，不消耗任何变量）
let s1 = String::from("Hello");
let s2 = String::from("World");
let s3 = format!("{} {}!", s1, s2);
println!("{}", s1);  // ✅ 仍可用
```

## 大坑：不能用索引访问！

```rust
let s = String::from("hello");
// let c = s[0];  // ❌ 编译错误！

// 为什么？因为 Rust 字符串是 UTF-8 编码
let chinese = String::from("你好");
println!("{}", chinese.len());  // 6，不是 2！

// 每个汉字占 3 个字节
```

## 正确的遍历和切片

```rust
let s = String::from("你好Rust");

// 按字符遍历
for c in s.chars() {
    print!("{} ", c);  // 你 好 R u s t
}

// 按字节遍历
for b in s.bytes() {
    print!("{} ", b);  // 228 189 160 ...
}

// 切片（必须按 UTF-8 边界！）
let slice = &s[0..6];  // ✅ "你好"（6字节=2汉字）
// let bad = &s[0..1];  // ❌ panic! 切到汉字中间

// 安全取第 n 个字符
let third_char = s.chars().nth(2);  // Some('R')
```

## 常用方法速览

```rust
let s = String::from("  Hello, Rust!  ");

s.len()              // 16（字节数）
s.is_empty()         // false
s.trim()             // "Hello, Rust!"
s.to_uppercase()     // "  HELLO, RUST!  "
s.to_lowercase()     // "  hello, rust!  "
s.contains("Rust")   // true
s.replace("Rust", "World")  // "  Hello, World!  "
s.starts_with("  H") // true
s.ends_with("!  ")   // true
s.split(',')         // 返回迭代器
s.split_whitespace() // 按空白分割
```

## 今日小结

| 要点 | 记住 |
|------|------|
| `&str` vs `String` | 切片 vs 可增长堆字符串 |
| `+` 运算符 | 消耗左边的 String |
| `format!` | 最灵活，不消耗 |
| **不能用索引** | UTF-8 编码，字节 ≠ 字符 |
| 切片要小心 | 必须在字符边界 |
| `.chars()` | 按字符遍历 |

---

**下节课**：HashMap
