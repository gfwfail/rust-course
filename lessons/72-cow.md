# 第 72 课：Cow — 写时克隆的智能指针

> 日期：2026-02-21  
> 主题：Clone-on-Write 的艺术

---

## 📍 什么是 Cow？

```rust
use std::borrow::Cow;

enum Cow<'a, B: ?Sized + ToOwned> {
    Borrowed(&'a B),      // 借用的引用
    Owned(<B as ToOwned>::Owned),  // 拥有的值
}
```

**Cow = Clone on Write（写时克隆）**

核心思想：
- 如果只需要**读**，就用借用（零拷贝）
- 只有需要**写/修改**时，才克隆成拥有的值

---

## 🔧 基础用法

```rust
use std::borrow::Cow;

fn main() {
    // 情况1：借用 —— 不拷贝
    let borrowed: Cow<str> = Cow::Borrowed("hello");
    println!("借用: {}", borrowed);
    
    // 情况2：拥有 —— 已经是 owned
    let owned: Cow<str> = Cow::Owned(String::from("world"));
    println!("拥有: {}", owned);
}
```

---

## 💡 为什么需要 Cow？

经典场景：**函数可能返回原数据，也可能返回修改后的新数据**

```rust
use std::borrow::Cow;

// 去除字符串两端空格
// 如果没空格 —— 返回原引用（零拷贝）
// 如果有空格 —— 返回新 String
fn trim_spaces(s: &str) -> Cow<str> {
    let trimmed = s.trim();
    if trimmed.len() == s.len() {
        // 没变化，直接返回借用
        Cow::Borrowed(s)
    } else {
        // 有变化，返回新字符串
        Cow::Owned(trimmed.to_string())
    }
}

fn main() {
    let s1 = "hello";      // 没空格
    let s2 = "  world  ";  // 有空格
    
    let r1 = trim_spaces(s1);
    let r2 = trim_spaces(s2);
    
    println!("r1 is borrowed: {}", matches!(r1, Cow::Borrowed(_))); // true
    println!("r2 is owned: {}", matches!(r2, Cow::Owned(_)));       // true
}
```

用 PHP 类比：
```php
function trimSpaces(string $s): string {
    return trim($s);  // PHP 每次都返回新字符串
}
```

但 Rust 的 Cow 让你**避免不必要的拷贝**！

---

## 🎯 to_mut() — 写时克隆的关键

```rust
use std::borrow::Cow;

fn main() {
    let mut cow: Cow<str> = Cow::Borrowed("hello");
    
    // 调用 to_mut() 触发克隆（如果是 Borrowed）
    cow.to_mut().make_ascii_uppercase();
    
    println!("{}", cow);  // HELLO
    // 现在 cow 是 Owned 了
}
```

**to_mut() 的魔法：**
- 如果是 `Borrowed` → 克隆成 `Owned`，返回 `&mut`
- 如果已经是 `Owned` → 直接返回 `&mut`（不再克隆）

---

## 🔥 实战：配置默认值

```rust
use std::borrow::Cow;
use std::collections::HashMap;

struct Config {
    values: HashMap<String, String>,
}

impl Config {
    // 获取配置值，没有就返回默认值
    fn get_or_default<'a>(&'a self, key: &str, default: &'a str) -> Cow<'a, str> {
        match self.values.get(key) {
            Some(v) => Cow::Borrowed(v),      // 借用已有值
            None => Cow::Borrowed(default),   // 借用默认值
        }
    }
    
    // 获取配置值，可能需要处理
    fn get_processed(&self, key: &str) -> Cow<str> {
        match self.values.get(key) {
            Some(v) if v.contains("$HOME") => {
                // 需要处理 —— 克隆
                Cow::Owned(v.replace("$HOME", "/home/user"))
            }
            Some(v) => Cow::Borrowed(v),  // 不需要处理 —— 借用
            None => Cow::Borrowed(""),
        }
    }
}
```

---

## 📦 Cow 常用方法

```rust
use std::borrow::Cow;

fn main() {
    let mut cow: Cow<str> = Cow::Borrowed("hello");
    
    // is_borrowed() / is_owned()
    println!("借用的? {}", cow.is_borrowed()); // true
    
    // into_owned() — 消费 Cow，返回 Owned 值
    let owned: String = cow.into_owned();
    println!("{}", owned);
    
    // 从头来
    let cow: Cow<str> = Cow::Borrowed("world");
    
    // as_ref() — 获取 &str
    let s: &str = cow.as_ref();
    println!("{}", s);
}
```

---

## ⚡ Cow 用于函数参数

函数同时接受 `&str` 和 `String`：

```rust
use std::borrow::Cow;

fn greet(name: Cow<str>) {
    println!("Hello, {}!", name);
}

fn main() {
    // 传借用
    greet(Cow::Borrowed("Alice"));
    
    // 传 owned
    greet(Cow::Owned(String::from("Bob")));
    
    // 更优雅：用 Into<Cow>
    greet("Charlie".into());
    greet(String::from("Dave").into());
}
```

**更简洁的写法 — 泛型约束：**

```rust
fn greet<'a>(name: impl Into<Cow<'a, str>>) {
    let name = name.into();
    println!("Hello, {}!", name);
}

fn main() {
    greet("Alice");                    // &str
    greet(String::from("Bob"));        // String
}
```

---

## 🎭 Cow 不只用于字符串

```rust
use std::borrow::Cow;

fn maybe_clone_vec(v: &[i32], need_sort: bool) -> Cow<[i32]> {
    if need_sort {
        let mut sorted = v.to_vec();
        sorted.sort();
        Cow::Owned(sorted)
    } else {
        Cow::Borrowed(v)
    }
}

fn main() {
    let data = vec![3, 1, 2];
    
    let result1 = maybe_clone_vec(&data, false);
    println!("不排序: {:?}, borrowed={}", result1, result1.is_borrowed());
    
    let result2 = maybe_clone_vec(&data, true);
    println!("排序后: {:?}, owned={}", result2, result2.is_owned());
}
```

---

## 🧠 Cow 的内存布局

```
Cow::Borrowed(&str):
┌─────────┬──────────────┐
│ tag: 0  │ ptr + len    │  (16 bytes on 64-bit)
└─────────┴──────────────┘

Cow::Owned(String):
┌─────────┬─────────────────────┐
│ tag: 1  │ ptr + len + cap     │  (24 bytes + heap)
└─────────┴─────────────────────┘
```

Cow 是枚举，运行时通过 tag 区分是借用还是拥有。

---

## 🚀 性能优化场景

### 场景 1：解析配置文件

```rust
use std::borrow::Cow;

fn parse_value(input: &str) -> Cow<str> {
    // 大多数值不需要转义
    if !input.contains('\\') {
        return Cow::Borrowed(input);
    }
    
    // 只有含转义符的才需要处理
    let mut result = String::with_capacity(input.len());
    let mut chars = input.chars().peekable();
    
    while let Some(c) = chars.next() {
        if c == '\\' {
            if let Some(&next) = chars.peek() {
                chars.next();
                match next {
                    'n' => result.push('\n'),
                    't' => result.push('\t'),
                    _ => result.push(next),
                }
            }
        } else {
            result.push(c);
        }
    }
    
    Cow::Owned(result)
}
```

### 场景 2：URL 编码

```rust
use std::borrow::Cow;

fn url_encode(input: &str) -> Cow<str> {
    // 大部分 URL 不需要编码
    if input.chars().all(|c| c.is_ascii_alphanumeric() || "-_.~".contains(c)) {
        return Cow::Borrowed(input);
    }
    
    // 需要编码的情况
    let mut result = String::new();
    for c in input.chars() {
        if c.is_ascii_alphanumeric() || "-_.~".contains(c) {
            result.push(c);
        } else {
            for b in c.to_string().bytes() {
                result.push_str(&format!("%{:02X}", b));
            }
        }
    }
    
    Cow::Owned(result)
}
```

---

## ⚠️ 常见陷阱

### 陷阱 1：不必要地调用 to_owned()

```rust
// ❌ 错误：总是克隆
fn process(s: &str) -> String {
    s.to_owned()  // 即使不需要修改也克隆了
}

// ✅ 正确：用 Cow
fn process(s: &str) -> Cow<str> {
    if needs_modification(s) {
        Cow::Owned(modify(s))
    } else {
        Cow::Borrowed(s)
    }
}
```

### 陷阱 2：生命周期搞不定

```rust
// ❌ 编译错误：生命周期不匹配
fn bad<'a>(s: &'a str) -> Cow<'static, str> {
    Cow::Borrowed(s)  // 's' 的生命周期是 'a，不是 'static
}

// ✅ 正确：保持生命周期一致
fn good<'a>(s: &'a str) -> Cow<'a, str> {
    Cow::Borrowed(s)
}
```

---

## 🧠 本课要点

1. **Cow = Clone on Write** —— 延迟克隆，按需分配
2. **Borrowed** 时零拷贝，**Owned** 时才分配内存
3. **to_mut()** 触发克隆（如果需要）
4. 适合"大多数情况不修改，少数情况需要修改"的场景
5. 不只用于字符串 —— `Cow<[T]>`、`Cow<Path>` 等都可以

---

## 📝 练习思考

1. 为什么 `Cow` 需要 `ToOwned` trait？
   - 答案：因为需要从 `&B` 克隆出 `B::Owned`

2. `Cow<str>` 和 `Cow<String>` 有什么区别？
   - 答案：`Cow<str>` 借用 `&str`，拥有 `String`；`Cow<String>` 是错误的用法，应该用 `Cow<str>`

3. 什么时候用 Cow？
   - 答案：函数可能返回输入数据，也可能返回修改后的新数据；需要避免不必要的内存分配

---

## 📚 相关文档

- [std::borrow::Cow](https://doc.rust-lang.org/std/borrow/enum.Cow.html)
- [ToOwned trait](https://doc.rust-lang.org/std/borrow/trait.ToOwned.html)
- [Borrow and AsRef](https://doc.rust-lang.org/book/ch15-02-deref.html)

---

*下节课预告：PhantomData — 幽灵数据与零大小类型*
