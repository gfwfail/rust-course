# 第 108 课：std::char — Unicode 字符的秘密

## 📌 什么是 `char`？

Rust 的 `char` 是一个 **Unicode 标量值**（Unicode Scalar Value），占用 **4 字节**。

```rust
// char 是 Unicode 标量值，不是 ASCII
let a: char = 'a';         // ASCII 字母
let emoji: char = '🦀';    // Emoji 也是一个 char
let chinese: char = '中';  // 中文字符

// 每个 char 占 4 字节
println!("char 大小: {} bytes", std::mem::size_of::<char>()); // 4
```

对比 PHP/JS：
- PHP 没有真正的 char 类型，字符就是长度为 1 的字符串
- JS 的字符串是 UTF-16 编码，emoji 可能占 2 个 "字符"
- Rust 的 char 保证是单个 Unicode 标量值

---

## 🔍 char 的实用方法

### 1️⃣ 字符分类

```rust
let c = 'A';

// 判断字符类型
c.is_alphabetic()    // true - 是否为字母
c.is_numeric()       // false - 是否为数字 (0-9)
c.is_alphanumeric()  // true - 是否为字母或数字
c.is_whitespace()    // false - 是否为空白字符
c.is_ascii()         // true - 是否为 ASCII 字符

// 更细分的 ASCII 判断
c.is_ascii_alphabetic()   // true
c.is_ascii_uppercase()    // true
c.is_ascii_lowercase()    // false
c.is_ascii_digit()        // false - 是否为 ASCII 数字
c.is_ascii_hexdigit()     // true - A-F 也是十六进制数字
c.is_ascii_punctuation()  // false - 标点符号
c.is_ascii_control()      // false - 控制字符
```

### 2️⃣ 大小写转换

```rust
let c = 'a';

// 转换大小写（返回新 char）
let upper = c.to_ascii_uppercase();  // 'A'
let lower = 'Z'.to_ascii_lowercase(); // 'z'

// 注意：Unicode 大小写可能产生多个字符！
// 所以返回的是迭代器
let german_sharp_s = 'ß';
for c in german_sharp_s.to_uppercase() {
    print!("{}", c);  // 输出: SS（两个字符！）
}
```

### 3️⃣ 数字转换

```rust
// char 转数字
let c = '7';
let digit = c.to_digit(10);  // Some(7)
let hex = 'F'.to_digit(16);  // Some(15)
let invalid = 'G'.to_digit(16);  // None

// 数字转 char
let c = char::from_digit(10, 16);  // Some('a')
let c = char::from_digit(5, 10);   // Some('5')
```

---

## 🎯 char 与 Unicode 码点

```rust
// char 可以和 u32 互转
let c = '🦀';

// char → u32 (Unicode 码点)
let code_point: u32 = c as u32;  // 129408
let code_point = c as u32;       // 同上

// u32 → char (可能失败，因为不是所有 u32 都是有效 Unicode)
let c = char::from_u32(129408);  // Some('🦀')
let invalid = char::from_u32(0xD800);  // None (代理对，无效)

// 如果你确定是有效的，可以用 unsafe
let c = unsafe { char::from_u32_unchecked(65) };  // 'A'

// 转义序列
let c = '\u{1F980}';  // 🦀 (Unicode 转义)
let c = '\x41';       // 'A' (ASCII 转义)
```

---

## 💡 实战：字符验证器

```rust
/// 检查是否为有效的用户名字符
fn is_valid_username_char(c: char) -> bool {
    c.is_ascii_alphanumeric() || c == '_' || c == '-'
}

/// 检查是否为有效密码字符（必须包含各类字符）
fn analyze_password_char(c: char) -> &'static str {
    if c.is_ascii_uppercase() {
        "大写字母"
    } else if c.is_ascii_lowercase() {
        "小写字母"
    } else if c.is_ascii_digit() {
        "数字"
    } else if c.is_ascii_punctuation() {
        "特殊字符"
    } else if c.is_whitespace() {
        "空白字符（不推荐）"
    } else {
        "其他 Unicode 字符"
    }
}

fn main() {
    let username = "rust_crab-01";
    let valid = username.chars().all(is_valid_username_char);
    println!("用户名有效: {}", valid);  // true
    
    let password = "Rust@2026";
    for c in password.chars() {
        println!("'{}' → {}", c, analyze_password_char(c));
    }
}
```

---

## ⚠️ 注意：char 不等于 "字符"

这是个常见误区：

```rust
// 这个 emoji 在 Rust 中是多少个 char？
let family = "👨‍👩‍👧";  // 家庭 emoji

// 答案可能出乎意料
println!("字节数: {}", family.len());  // 18
println!("char 数: {}", family.chars().count());  // 5

// 为什么是 5 个 char？
// 因为 👨‍👩‍👧 是由多个 Unicode 标量值 + 零宽连接符组成的
for c in family.chars() {
    println!("{:?} (U+{:04X})", c, c as u32);
}
// 👨 (U+1F468) - 男人
// \u{200d}    - 零宽连接符 ZWJ
// 👩 (U+1F469) - 女人
// \u{200d}    - 零宽连接符 ZWJ
// 👧 (U+1F467) - 女孩
```

如果要正确处理"人类视觉上的字符"（grapheme clusters），需要用第三方库 `unicode-segmentation`。

---

## 🔧 char 常用方法速查

| 方法 | 说明 | 示例 |
|------|------|------|
| `is_alphabetic()` | 是否为字母 | `'中'.is_alphabetic()` → `true` |
| `is_numeric()` | 是否为数字 | `'①'.is_numeric()` → `true` |
| `is_ascii()` | 是否为 ASCII | `'a'.is_ascii()` → `true` |
| `is_ascii_digit()` | 是否为 0-9 | `'5'.is_ascii_digit()` → `true` |
| `to_digit(radix)` | 转为数字 | `'F'.to_digit(16)` → `Some(15)` |
| `to_ascii_uppercase()` | 转大写 | `'a'.to_ascii_uppercase()` → `'A'` |
| `escape_unicode()` | Unicode 转义 | `'🦀'.escape_unicode()` → `\u{1f980}` |
| `len_utf8()` | UTF-8 编码长度 | `'🦀'.len_utf8()` → `4` |

---

## 📝 小结

1. **char 是 4 字节**的 Unicode 标量值，不是字节
2. **丰富的分类方法**：`is_alphabetic()`、`is_ascii_*()` 等
3. **大小写转换**可能产生多个字符（Unicode 特性）
4. **char 不等于视觉字符**：emoji 可能由多个 char 组成
5. **与 u32 互转**：`as u32` 和 `char::from_u32()`

---

*发布日期：2026-02-28*
