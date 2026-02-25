# 第 100 课：std::error — Rust 错误处理完整指南 🎉

恭喜来到第 100 课！今天深入讲解 Rust 的错误处理机制，这是 Rust 区别于其他语言的核心特性之一。

---

## 🎯 Rust 错误处理哲学

```rust
// ❌ 其他语言：异常（exception）
// 问题：异常可以从任何地方抛出，调用者可能忘记处理

// ✅ Rust：Result<T, E>
// 错误是返回值的一部分，编译器强制你处理
fn read_file(path: &str) -> Result<String, std::io::Error> {
    std::fs::read_to_string(path)
}

// 不处理 Result？编译器警告你！
```

**核心思想**：让错误处理成为类型系统的一部分，无法忽略。

---

## 📦 Result 和 Option 回顾

```rust
// Option — 值可能不存在
enum Option<T> {
    Some(T),
    None,
}

// Result — 操作可能失败
enum Result<T, E> {
    Ok(T),
    Err(E),
}

// 两者的对应关系
let opt: Option<i32> = Some(42);
let res: Result<i32, ()> = opt.ok_or(());  // Option → Result

let res: Result<i32, &str> = Ok(42);
let opt: Option<i32> = res.ok();           // Result → Option
```

---

## 🔧 ? 操作符 — 错误传播神器

```rust
use std::fs::File;
use std::io::{self, Read};

// ❌ 手动处理错误（繁琐）
fn read_file_v1(path: &str) -> Result<String, io::Error> {
    let file = File::open(path);
    let mut file = match file {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    
    let mut contents = String::new();
    match file.read_to_string(&mut contents) {
        Ok(_) => Ok(contents),
        Err(e) => Err(e),
    }
}

// ✅ 用 ? 操作符（优雅）
fn read_file_v2(path: &str) -> Result<String, io::Error> {
    let mut file = File::open(path)?;  // 失败则提前返回 Err
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}

// ✅ 链式调用更简洁
fn read_file_v3(path: &str) -> Result<String, io::Error> {
    let mut contents = String::new();
    File::open(path)?.read_to_string(&mut contents)?;
    Ok(contents)
}
```

### ? 的本质

```rust
// file? 等价于：
match file {
    Ok(v) => v,
    Err(e) => return Err(e.into()),  // 注意：会自动调用 into()！
}
```

---

## 🎨 std::error::Error trait

这是 Rust 错误类型的核心 trait：

```rust
pub trait Error: Debug + Display {
    // 错误的来源（可选）
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

### 实现自定义错误

```rust
use std::error::Error;
use std::fmt;

// 1. 定义错误类型
#[derive(Debug)]
struct MyError {
    message: String,
    source: Option<Box<dyn Error + 'static>>,
}

// 2. 实现 Display（给用户看的信息）
impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.message)
    }
}

// 3. 实现 Error trait
impl Error for MyError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        self.source.as_ref().map(|e| e.as_ref())
    }
}

// 4. 使用
fn do_something() -> Result<(), MyError> {
    Err(MyError {
        message: "出错了！".to_string(),
        source: None,
    })
}
```

---

## 📚 错误类型设计模式

### 模式一：枚举错误（推荐）

```rust
use std::{io, num::ParseIntError};

#[derive(Debug)]
enum ConfigError {
    IoError(io::Error),
    ParseError(ParseIntError),
    MissingField(String),
    InvalidValue { field: String, value: String },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::IoError(e) => write!(f, "IO 错误: {}", e),
            ConfigError::ParseError(e) => write!(f, "解析错误: {}", e),
            ConfigError::MissingField(name) => write!(f, "缺少字段: {}", name),
            ConfigError::InvalidValue { field, value } => {
                write!(f, "无效值: {} = {}", field, value)
            }
        }
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::IoError(e) => Some(e),
            ConfigError::ParseError(e) => Some(e),
            _ => None,
        }
    }
}

// From trait 实现自动转换
impl From<io::Error> for ConfigError {
    fn from(err: io::Error) -> Self {
        ConfigError::IoError(err)
    }
}

impl From<ParseIntError> for ConfigError {
    fn from(err: ParseIntError) -> Self {
        ConfigError::ParseError(err)
    }
}

// 现在 ? 可以自动转换错误类型了！
fn load_config(path: &str) -> Result<Config, ConfigError> {
    let content = std::fs::read_to_string(path)?;  // io::Error → ConfigError
    let port: u16 = content.trim().parse()?;       // ParseIntError → ConfigError
    Ok(Config { port })
}
```

### 模式二：Box<dyn Error>（快速原型）

```rust
// 当你不想定义具体错误类型时
type Result<T> = std::result::Result<T, Box<dyn Error>>;

fn quick_and_dirty() -> Result<()> {
    let content = std::fs::read_to_string("config.txt")?;
    let num: i32 = content.trim().parse()?;
    Ok(())
}

// 缺点：丢失了具体错误类型信息，无法 match
```

---

## 🔗 错误链追踪

```rust
use std::error::Error;

fn print_error_chain(err: &dyn Error) {
    println!("错误: {}", err);
    
    let mut source = err.source();
    while let Some(cause) = source {
        println!("  原因: {}", cause);
        source = cause.source();
    }
}

// 使用示例
fn main() {
    if let Err(e) = run() {
        print_error_chain(&*e);
    }
}

// 输出可能是：
// 错误: 配置加载失败
//   原因: 无法读取文件
//   原因: No such file or directory (os error 2)
```

---

## 🎭 常见处理策略

### 1. 传播错误（最常见）

```rust
fn process() -> Result<Data, Error> {
    let input = read_input()?;
    let data = parse(input)?;
    Ok(data)
}
```

### 2. 提供默认值

```rust
let config = load_config().unwrap_or_default();
let port = parse_port().unwrap_or(8080);
```

### 3. 记录并忽略

```rust
if let Err(e) = optional_cleanup() {
    eprintln!("警告: 清理失败: {}", e);
    // 继续执行
}
```

### 4. 转换错误类型

```rust
let data = parse_json(input)
    .map_err(|e| MyError::new(format!("JSON 解析失败: {}", e)))?;
```

### 5. panic（仅用于不可恢复的错误）

```rust
// 配置文件必须存在，否则程序无法运行
let config = load_config().expect("配置文件缺失！");
```

---

## ⚡ unwrap vs expect vs ?

```rust
// unwrap — 失败则 panic，信息不友好
let file = File::open("data.txt").unwrap();

// expect — 失败则 panic，有自定义信息
let file = File::open("data.txt").expect("无法打开 data.txt");

// ? — 传播错误，由调用者决定如何处理（推荐！）
let file = File::open("data.txt")?;
```

**原则**：
- 库代码：永远用 `?`，让调用者决定
- 应用代码：必须成功的操作用 `expect`，可选操作用 `?`

---

## 🧪 main 函数的 Result

```rust
// main 可以返回 Result！
fn main() -> Result<(), Box<dyn Error>> {
    let content = std::fs::read_to_string("input.txt")?;
    println!("{}", content);
    Ok(())
}

// 程序会在 Err 时退出，并打印错误信息
```

---

## 💡 实战：配置解析器

```rust
use std::collections::HashMap;
use std::error::Error;
use std::fmt;
use std::fs;

#[derive(Debug)]
enum ConfigError {
    Io(std::io::Error),
    Parse { line: usize, message: String },
    Missing(String),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Io(e) => write!(f, "读取配置失败: {}", e),
            ConfigError::Parse { line, message } => {
                write!(f, "第 {} 行解析错误: {}", line, message)
            }
            ConfigError::Missing(key) => write!(f, "缺少配置项: {}", key),
        }
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::Io(e) => Some(e),
            _ => None,
        }
    }
}

impl From<std::io::Error> for ConfigError {
    fn from(err: std::io::Error) -> Self {
        ConfigError::Io(err)
    }
}

fn parse_config(path: &str) -> Result<HashMap<String, String>, ConfigError> {
    let content = fs::read_to_string(path)?;
    let mut config = HashMap::new();
    
    for (i, line) in content.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        
        let parts: Vec<&str> = line.splitn(2, '=').collect();
        if parts.len() != 2 {
            return Err(ConfigError::Parse {
                line: i + 1,
                message: "格式应为 key=value".to_string(),
            });
        }
        
        config.insert(parts[0].trim().to_string(), parts[1].trim().to_string());
    }
    
    Ok(config)
}

fn get_required(config: &HashMap<String, String>, key: &str) -> Result<&str, ConfigError> {
    config.get(key)
        .map(|s| s.as_str())
        .ok_or_else(|| ConfigError::Missing(key.to_string()))
}
```

---

## 🎓 第 100 课小结

| 概念 | 说明 |
|------|------|
| `Result<T, E>` | 可能失败的操作返回类型 |
| `?` 操作符 | 错误传播，失败时提前返回 |
| `Error` trait | 错误类型的标准接口 |
| `From` trait | 实现错误类型自动转换 |
| `source()` | 获取错误链的根本原因 |

**核心原则**：
1. 用类型系统表达可能的失败
2. 强制调用者处理错误
3. 错误信息要对人友好
4. 保留错误链便于调试

---

## 🎉 里程碑达成！

恭喜完成 100 节 Rust 课程！到目前为止我们已经学习了：
- Rust 基础语法和核心概念
- 所有权、借用、生命周期
- 各种 trait 和泛型
- 标准库的主要模块

接下来我们会继续深入更多高级主题。保持学习！💪

---

*第 100 课完*
