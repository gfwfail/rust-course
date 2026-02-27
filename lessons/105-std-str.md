# 第 105 课：std::str — 字符串切片方法大全

## 字符串切片 vs String

先搞清楚 Rust 中字符串的两兄弟：

```rust
let s1: &str = "hello";        // 字符串切片（借用）
let s2: String = String::from("hello");  // 堆上的字符串（拥有）

// 类比 PHP：
// &str ≈ 字符串字面量（只读引用）
// String ≈ 动态分配的字符串
```

`&str` 是 UTF-8 编码的字节切片，今天专门讲它的方法。

---

## 一、基础查询方法

```rust
let s = "Hello, 世界!";

// 长度（字节数，不是字符数！）
s.len()           // → 14 (不是 9)
s.is_empty()      // → false

// 字符数
s.chars().count() // → 9
```

⚠️ **PHP 坑警告**：
```php
strlen("世界");  // PHP: 6 (字节)
mb_strlen("世界"); // PHP: 2 (字符)
```
Rust 的 `len()` 是字节数，用 `chars().count()` 得字符数。

---

## 二、字符与字节遍历

```rust
let s = "Rust 🦀";

// 按字符遍历
for c in s.chars() {
    println!("{c}");  // R, u, s, t, ' ', 🦀
}

// 按字节遍历
for b in s.bytes() {
    println!("{b:02x}");  // 52, 75, 73, 74, 20, f0, 9f, a6, 80
}

// 带索引遍历（字节索引）
for (i, c) in s.char_indices() {
    println!("{i}: {c}");
    // 0: R, 1: u, 2: s, 3: t, 4: ' ', 5: 🦀
}
```

---

## 三、搜索方法

### contains / starts_with / ends_with

```rust
let s = "hello world";

s.contains("world")      // → true
s.starts_with("hello")   // → true
s.ends_with("world")     // → true

// 支持 char
s.contains('o')          // → true
```

### find / rfind — 找位置

```rust
let s = "hello hello";

s.find("lo")        // → Some(3)  第一次出现的字节位置
s.rfind("lo")       // → Some(9)  最后一次出现

s.find('x')         // → None
```

### matches / match_indices — 找所有匹配

```rust
let s = "abcabc";

// 找所有匹配
let v: Vec<_> = s.matches("bc").collect();
// → ["bc", "bc"]

// 找所有匹配及其位置
let v: Vec<_> = s.match_indices("bc").collect();
// → [(1, "bc"), (4, "bc")]
```

---

## 四、分割方法

### split 系列

```rust
let s = "a,b,c,d";

// 基础分割
let parts: Vec<&str> = s.split(',').collect();
// → ["a", "b", "c", "d"]

// 限制分割次数
let parts: Vec<_> = s.splitn(2, ',').collect();
// → ["a", "b,c,d"]

// 从右边开始分割
let parts: Vec<_> = s.rsplitn(2, ',').collect();
// → ["d", "a,b,c"]

// 用闭包分割
let s = "abc123def456";
let parts: Vec<_> = s.split(|c: char| c.is_numeric()).collect();
// → ["abc", "", "", "def", "", "", ""]
```

### split_whitespace — 按空白分割

```rust
let s = "  hello   world  ";

// split_whitespace 自动忽略连续空白
let words: Vec<_> = s.split_whitespace().collect();
// → ["hello", "world"]

// 对比普通 split
let parts: Vec<_> = s.split(' ').collect();
// → ["", "", "hello", "", "", "world", "", ""]
```

### lines — 按行分割

```rust
let text = "line1\nline2\r\nline3";

for line in text.lines() {
    println!("{line}");
}
// 自动处理 \n 和 \r\n
```

---

## 五、修剪方法 (trim)

```rust
let s = "  hello  ";

s.trim()        // → "hello"     // 两端空白
s.trim_start()  // → "hello  "   // 左边
s.trim_end()    // → "  hello"   // 右边

// 修剪指定字符
let s = "xxxhelloyyy";
s.trim_matches('x')       // → "helloyyy"
s.trim_start_matches('x') // → "helloyyy"
s.trim_matches(|c| c == 'x' || c == 'y')  // → "hello"
```

---

## 六、大小写转换

```rust
let s = "Hello World";

s.to_lowercase()  // → "hello world" (返回 String)
s.to_uppercase()  // → "HELLO WORLD"
s.to_ascii_lowercase() // 仅 ASCII

// 首字母大写没有内置方法，需要自己写
fn capitalize(s: &str) -> String {
    let mut chars = s.chars();
    match chars.next() {
        None => String::new(),
        Some(c) => c.to_uppercase().chain(chars).collect(),
    }
}
```

---

## 七、替换方法

```rust
let s = "hello hello";

s.replace("hello", "hi")      // → "hi hi"
s.replacen("hello", "hi", 1)  // → "hi hello"（只替换1次）
```

---

## 八、解析方法 — parse()

这是个神器，可以把字符串解析成任何实现了 `FromStr` 的类型：

```rust
let s = "42";

let n: i32 = s.parse().unwrap();  // → 42
let n: f64 = s.parse().unwrap();  // → 42.0
let n = s.parse::<i32>().unwrap(); // turbofish 写法

// 处理错误
let result: Result<i32, _> = "abc".parse();
// → Err(ParseIntError)

// 实际用法
fn read_port(s: &str) -> Option<u16> {
    s.parse().ok()
}
```

---

## 九、实战模式

### 1. 解析 key=value

```rust
fn parse_pair(s: &str) -> Option<(&str, &str)> {
    let (key, value) = s.split_once('=')?;
    Some((key.trim(), value.trim()))
}

parse_pair("name = Alice")  // → Some(("name", "Alice"))
parse_pair("invalid")       // → None
```

### 2. 提取域名

```rust
fn extract_domain(url: &str) -> Option<&str> {
    let url = url.strip_prefix("https://")
        .or_else(|| url.strip_prefix("http://"))?;
    url.split('/').next()
}

extract_domain("https://example.com/path")
// → Some("example.com")
```

### 3. 安全截取（按字符）

```rust
fn safe_truncate(s: &str, max_chars: usize) -> &str {
    match s.char_indices().nth(max_chars) {
        Some((idx, _)) => &s[..idx],
        None => s,
    }
}

safe_truncate("Hello 世界", 7)  // → "Hello 世"
```

---

## 十、方法速查表

| 方法 | 作用 |
|------|------|
| `len()` | 字节长度 |
| `is_empty()` | 是否为空 |
| `contains()` | 包含子串 |
| `starts_with()` | 前缀匹配 |
| `ends_with()` | 后缀匹配 |
| `find()` / `rfind()` | 查找位置 |
| `split()` | 分割 |
| `split_once()` | 分割一次 |
| `lines()` | 按行分割 |
| `trim()` | 修剪空白 |
| `replace()` | 替换 |
| `parse()` | 解析成其他类型 |
| `to_lowercase()` | 转小写 |
| `chars()` | 字符迭代器 |
| `bytes()` | 字节迭代器 |

---

## 课后思考

为什么 `"hello"[0]` 在 Rust 里编译不过？

答案：因为 Rust 的字符串是 UTF-8 编码，一个字符可能占多个字节。`[0]` 这种索引操作期望是 O(1)，但 UTF-8 不是固定宽度编码，所以 Rust 禁止这种操作，强制你用 `.chars().nth(0)` 明确表达意图。

---

下节预告：**std::slice — 切片操作的艺术** 🦀
