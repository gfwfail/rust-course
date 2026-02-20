# 第 66 课：类型转换 (Type Conversions)

> 日期：2026-02-21  
> 主题：From, Into, TryFrom, TryInto, AsRef, AsMut

---

## 为什么重要？

PHP/JS 是弱类型，类型转换经常是隐式的：

```php
$num = "42";
echo $num + 1;  // 43，自动转
```

Rust 是强类型，**所有转换必须显式**。这避免了 bug，但也意味着你得掌握正确的转换方式。

---

## Rust 的类型转换体系

```
                    可能失败？
                    ↓
              ┌─────┴─────┐
              │           │
           不会失败     可能失败
              │           │
              ▼           ▼
         From/Into    TryFrom/TryInto
              │
              │ 只是借用？
              ▼
         AsRef/AsMut
```

---

## 1. From 与 Into（不会失败的转换）

### From — "从某类型创建"

```rust
// 标准库已实现
let s: String = String::from("hello");
let num: i64 = i64::from(42i32);

// 自定义类型
struct UserId(u64);

impl From<u64> for UserId {
    fn from(id: u64) -> Self {
        UserId(id)
    }
}

let user_id = UserId::from(12345);
```

### Into — "转换为某类型"

`Into` 是 `From` 的反向，**实现 From 自动获得 Into**：

```rust
// 因为实现了 From<u64> for UserId
// 自动得到 Into<UserId> for u64
let user_id: UserId = 12345u64.into();
```

### 经验法则

- **实现 `From`，不要实现 `Into`**
- Into 主要用于泛型约束

```rust
// 接受任何能转成 String 的参数
fn greet(name: impl Into<String>) {
    let name = name.into();
    println!("Hello, {}!", name);
}

greet("Alice");           // &str → String
greet(String::from("Bob")); // String → String（零成本）
```

---

## 2. TryFrom 与 TryInto（可能失败的转换）

当转换可能失败时，用 `Try` 版本，返回 `Result`：

```rust
use std::convert::TryFrom;

// i32 转 u8 可能溢出
let big: i32 = 300;
let result = u8::try_from(big);
// Err(TryFromIntError)

let small: i32 = 42;
let result = u8::try_from(small);
// Ok(42)
```

### 自定义 TryFrom

```rust
struct Port(u16);

#[derive(Debug)]
struct PortError;

impl TryFrom<i32> for Port {
    type Error = PortError;
    
    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value < 0 || value > 65535 {
            Err(PortError)
        } else {
            Ok(Port(value as u16))
        }
    }
}

// 使用
let port = Port::try_from(8080)?;  // Ok
let port = Port::try_from(-1)?;    // Err
```

---

## 3. AsRef 与 AsMut（借用转换）

不转移所有权，只是换个"视角"看数据：

```rust
// AsRef<str> 让函数接受 String 或 &str
fn print_len(s: impl AsRef<str>) {
    println!("Length: {}", s.as_ref().len());
}

print_len("hello");           // &str
print_len(String::from("hi")); // String

// AsRef<[u8]> 同理
fn hash_data(data: impl AsRef<[u8]>) {
    let bytes = data.as_ref();
    // ...
}

hash_data("hello");        // &str → &[u8]
hash_data(vec![1, 2, 3]);  // Vec<u8> → &[u8]
```

### AsRef vs Deref

- `Deref`: 智能指针用，提供 `*` 解引用
- `AsRef`: 显式借用转换，用于泛型约束

---

## 4. as 关键字（原始类型转换）

只用于**数值类型**之间的转换：

```rust
let x: i32 = 42;
let y: i64 = x as i64;  // 扩展，安全
let z: i16 = x as i16;  // 截断，可能丢数据！

let ptr = &x as *const i32;  // 引用转裸指针

// ⚠️ as 不做任何检查，可能截断
let big: u32 = 300;
let small: u8 = big as u8;  // 44（300 % 256）
```

**建议**：能用 `From`/`TryFrom` 就用，`as` 只用于你确定不会出问题的场景。

---

## 5. 实战：构建灵活的 API

```rust
use std::path::Path;

// 接受 &str、String、PathBuf、&Path...
fn read_config(path: impl AsRef<Path>) -> String {
    let path = path.as_ref();
    std::fs::read_to_string(path).unwrap()
}

read_config("config.toml");
read_config(String::from("/etc/app.conf"));
read_config(Path::new("settings.json"));
```

### 组合多个约束

```rust
fn process<T>(input: T) 
where
    T: AsRef<str> + Into<String>,
{
    // 可以借用
    println!("Processing: {}", input.as_ref());
    // 也可以获取所有权
    let owned: String = input.into();
}
```

---

## 对比 PHP/Laravel

```php
// PHP：手动写转换方法
class User {
    public static function fromArray(array $data): self { ... }
    public function toArray(): array { ... }
}
```

```rust
// Rust：统一的转换接口
impl From<HashMap<String, String>> for User {
    fn from(data: HashMap<String, String>) -> Self { ... }
}

// 自动获得 .into()
let user: User = data.into();
```

---

## 速查表

| 场景 | 用什么 |
|------|--------|
| 总是成功的转换 | `From` / `Into` |
| 可能失败的转换 | `TryFrom` / `TryInto` |
| 借用为另一种类型 | `AsRef` / `AsMut` |
| 原始数值类型转换 | `as`（谨慎使用） |
| 智能指针自动解引用 | `Deref` |

---

## 要点总结

1. **实现 `From`，自动获得 `Into`**
2. **可能失败就用 `Try` 版本**
3. **`AsRef` 让 API 更灵活**
4. **`as` 只用于原始类型，小心截断**
5. **用泛型约束写出灵活的函数签名**

---

## 下节预告

**Deref 与智能指针的魔法** — 为什么 `String` 可以当 `&str` 用？

---

*课程笔记：性奴001*
