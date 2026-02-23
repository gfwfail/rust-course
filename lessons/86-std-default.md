# 第 86 课：std::default — Default trait 与默认值

## 📌 为什么需要 Default？

在写代码时，我们经常需要一个「默认值」：

```rust
// PHP: 参数默认值
function createUser($name, $role = "guest") { ... }

// JS: 对象解构默认值
const { name, role = "guest" } = config;
```

Rust 没有函数参数默认值，但有一个更强大的机制：**Default trait**。

---

## 🎯 Default trait 定义

```rust
pub trait Default: Sized {
    fn default() -> Self;
}
```

就这么简单！实现 `default()` 方法，返回一个该类型的默认实例。

---

## 📦 标准库类型的 Default

几乎所有标准库类型都实现了 Default：

```rust
use std::collections::HashMap;

fn main() {
    // 数值类型 → 0
    let n: i32 = Default::default();    // 0
    let f: f64 = Default::default();    // 0.0
    
    // 布尔 → false
    let b: bool = Default::default();   // false
    
    // 字符串 → 空
    let s: String = Default::default(); // ""
    
    // 集合 → 空
    let v: Vec<i32> = Default::default();     // []
    let m: HashMap<String, i32> = Default::default(); // {}
    
    // Option → None
    let o: Option<i32> = Default::default();  // None
    
    println!("i32: {n}, f64: {f}, bool: {b}");
    println!("String: {:?}, Vec: {:?}", s, v);
}
```

---

## ✨ 自定义类型实现 Default

### 方式一：手动实现

```rust
struct Config {
    host: String,
    port: u16,
    debug: bool,
    max_connections: usize,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            host: String::from("localhost"),
            port: 8080,
            debug: false,
            max_connections: 100,
        }
    }
}

fn main() {
    let config = Config::default();
    println!("{}:{}", config.host, config.port);
    // localhost:8080
}
```

### 方式二：derive 自动派生

如果所有字段都实现了 Default，可以直接 derive：

```rust
#[derive(Default)]
struct Point {
    x: f64,  // → 0.0
    y: f64,  // → 0.0
    z: f64,  // → 0.0
}

fn main() {
    let origin = Point::default();
    println!("({}, {}, {})", origin.x, origin.y, origin.z);
    // (0, 0, 0)
}
```

---

## 🔥 实战技巧：部分覆盖 (Struct Update Syntax)

这是 Default 最常用的模式，类似 JS 的 spread operator：

```rust
#[derive(Default, Debug)]
struct ServerConfig {
    host: String,
    port: u16,
    threads: usize,
    timeout_secs: u64,
    debug: bool,
}

fn main() {
    // 只指定想修改的字段，其他用默认值
    let config = ServerConfig {
        port: 3000,
        debug: true,
        ..Default::default()  // 剩余字段用默认值填充
    };
    
    println!("{:#?}", config);
    // ServerConfig {
    //     host: "",
    //     port: 3000,
    //     threads: 0,
    //     timeout_secs: 0,
    //     debug: true,
    // }
}
```

**这就是 Rust 替代「函数参数默认值」的惯用法！**

---

## 🏗️ Builder 模式 + Default

Default 经常配合 Builder 模式使用：

```rust
#[derive(Default)]
struct RequestBuilder {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    timeout: u64,
}

impl RequestBuilder {
    fn new(url: impl Into<String>) -> Self {
        Self {
            url: url.into(),
            method: "GET".into(),
            ..Default::default()
        }
    }
    
    fn method(mut self, m: &str) -> Self {
        self.method = m.into();
        self
    }
    
    fn header(mut self, k: &str, v: &str) -> Self {
        self.headers.push((k.into(), v.into()));
        self
    }
    
    fn timeout(mut self, secs: u64) -> Self {
        self.timeout = secs;
        self
    }
}

fn main() {
    let req = RequestBuilder::new("https://api.example.com")
        .method("POST")
        .header("Content-Type", "application/json")
        .timeout(30);
        
    println!("{} {} (timeout: {}s)", req.method, req.url, req.timeout);
}
```

---

## 🎨 Enum 的 Default

枚举也可以实现 Default，用 `#[default]` 标记默认变体：

```rust
#[derive(Default, Debug)]
enum Status {
    #[default]
    Pending,
    Processing,
    Completed,
    Failed,
}

fn main() {
    let status: Status = Default::default();
    println!("{:?}", status); // Pending
}
```

---

## 💡 Default 在泛型中的妙用

### 例子：带默认值的 Option 解包

```rust
fn get_or_default<T: Default>(opt: Option<T>) -> T {
    opt.unwrap_or_default()  // 标准库方法！
}

fn main() {
    let some_num: Option<i32> = Some(42);
    let none_num: Option<i32> = None;
    
    println!("{}", get_or_default(some_num)); // 42
    println!("{}", get_or_default(none_num)); // 0
}
```

### 例子：HashMap 的 entry API

```rust
use std::collections::HashMap;

fn main() {
    let mut counts: HashMap<&str, i32> = HashMap::new();
    
    let words = ["apple", "banana", "apple", "cherry", "banana", "apple"];
    
    for word in words {
        // or_default() 使用 i32::default() = 0
        *counts.entry(word).or_default() += 1;
    }
    
    println!("{:?}", counts);
    // {"apple": 3, "banana": 2, "cherry": 1}
}
```

---

## 📝 Default vs new()

什么时候用哪个？

| 场景 | 推荐 |
|------|------|
| 无参数创建实例 | `Default::default()` |
| 需要必填参数 | `new(required_arg)` |
| 语义上是「空」或「零值」| `Default` |
| 语义上是「初始化」| `new()` |

```rust
// Vec: default() 和 new() 等价
let v1: Vec<i32> = Vec::new();
let v2: Vec<i32> = Vec::default();  // 完全一样

// 但语义不同的类型
struct Connection { addr: String }
impl Connection {
    // new 需要地址参数，不适合 Default
    fn new(addr: &str) -> Self { 
        Self { addr: addr.into() }
    }
}
```

---

## 🧩 小练习

实现一个 `GameSettings` 结构体：

```rust
#[derive(Debug)]
struct GameSettings {
    resolution: (u32, u32),  // 默认 1920x1080
    fullscreen: bool,        // 默认 false
    volume: u8,              // 默认 80
    difficulty: String,      // 默认 "normal"
}

// 实现 Default trait
impl Default for GameSettings {
    fn default() -> Self {
        Self {
            resolution: (1920, 1080),
            fullscreen: false,
            volume: 80,
            difficulty: String::from("normal"),
        }
    }
}

fn main() {
    // 玩家只想改分辨率
    let settings = GameSettings {
        resolution: (2560, 1440),
        ..Default::default()
    };
    println!("{:#?}", settings);
}
```

---

## 📌 本课要点

1. **Default trait** 提供类型的默认值
2. **derive(Default)** 可自动派生（所有字段需实现 Default）
3. **..Default::default()** 是 Rust 替代参数默认值的惯用法
4. **unwrap_or_default()** 和 **or_default()** 是常用的配套方法
5. Enum 用 `#[default]` 标记默认变体

Default 看似简单，但它是 Rust 生态中使用频率最高的 trait 之一！

---

下节课预告：**std::clone — Clone trait 深入**
