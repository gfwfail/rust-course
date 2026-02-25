# 第 100 课：std::error — Error Trait 与错误处理体系 🎉

第100课里程碑！今天深入讲 Rust 标准库中的 `Error` trait，这是整个错误处理体系的基石。

---

## 📦 Error trait 是什么？

```rust
// std::error::Error 的定义（简化版）
pub trait Error: Debug + Display {
    // 返回导致这个错误的底层错误（可选）
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

`Error` trait 有两个超级 trait 约束：
- `Debug` — 用于 `{:?}` 格式化
- `Display` — 用于 `{}` 格式化（给用户看的消息）

---

## 🔨 自定义错误类型

### 最简单的方式

```rust
use std::fmt;
use std::error::Error;

#[derive(Debug)]
struct MyError {
    message: String,
}

// 实现 Display（必须）
impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.message)
    }
}

// 实现 Error（可以用默认实现）
impl Error for MyError {}

fn might_fail() -> Result<(), MyError> {
    Err(MyError { 
        message: "出错了！".to_string() 
    })
}
```

---

## 🔗 错误链：source() 方法

当一个错误是由另一个错误导致的，用 `source()` 建立错误链：

```rust
use std::error::Error;
use std::fmt;
use std::io;

#[derive(Debug)]
struct ConfigError {
    message: String,
    source: io::Error,  // 底层 IO 错误
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "配置加载失败: {}", self.message)
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        Some(&self.source)  // 返回底层错误
    }
}

fn load_config() -> Result<(), ConfigError> {
    let io_err = io::Error::new(io::ErrorKind::NotFound, "文件不存在");
    Err(ConfigError {
        message: "config.toml".to_string(),
        source: io_err,
    })
}

fn main() {
    if let Err(e) = load_config() {
        println!("错误: {}", e);
        
        // 遍历错误链
        let mut source = e.source();
        while let Some(err) = source {
            println!("  原因: {}", err);
            source = err.source();
        }
    }
}
// 输出:
// 错误: 配置加载失败: config.toml
//   原因: 文件不存在
```

---

## 📊 std::error 的类型

标准库提供了一些内置错误类型：

```rust
use std::str::FromStr;
use std::num::ParseIntError;

// ParseIntError 实现了 Error
fn parse_number(s: &str) -> Result<i32, ParseIntError> {
    s.parse()
}

// 其他常见错误类型：
// std::io::Error          — IO 操作错误
// std::fmt::Error         — 格式化错误
// std::str::Utf8Error     — UTF-8 解码错误
// std::string::FromUtf8Error
// std::num::ParseIntError — 整数解析错误
// std::num::ParseFloatError
// std::env::VarError      — 环境变量错误
// std::array::TryFromSliceError
```

---

## 🎯 Box<dyn Error> — 通用错误类型

当函数可能返回多种错误类型时：

```rust
use std::error::Error;
use std::fs::File;
use std::io::Read;

// Box<dyn Error> 可以包装任何实现了 Error 的类型
fn read_config() -> Result<String, Box<dyn Error>> {
    let mut file = File::open("config.txt")?;  // io::Error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;        // io::Error
    
    // 如果需要解析数字
    let _port: u16 = contents.trim().parse()?;  // ParseIntError
    
    Ok(contents)
}
```

**类比 PHP/JS：** 就像 PHP 的 `Exception` 或 JS 的 `Error` 基类，`Box<dyn Error>` 是一个通用错误容器。

---

## 🔄 From trait 与错误转换

`?` 运算符自动调用 `From` 进行类型转换：

```rust
use std::error::Error;
use std::fmt;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "IO 错误: {}", e),
            AppError::Parse(e) => write!(f, "解析错误: {}", e),
        }
    }
}

impl Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            AppError::Io(e) => Some(e),
            AppError::Parse(e) => Some(e),
        }
    }
}

// 关键：实现 From trait
impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::Io(e)
    }
}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self {
        AppError::Parse(e)
    }
}

// 现在可以用 ? 自动转换
fn process() -> Result<(), AppError> {
    let content = std::fs::read_to_string("data.txt")?; // io::Error → AppError
    let _num: i32 = content.trim().parse()?;           // ParseIntError → AppError
    Ok(())
}
```

---

## 💡 核心要点总结

| 概念 | 用途 |
|------|------|
| `Error` trait | 错误类型的抽象接口 |
| `Display` | 给用户看的错误消息 |
| `Debug` | 给开发者调试用 |
| `source()` | 获取底层错误（错误链） |
| `Box<dyn Error>` | 通用错误容器 |
| `From` trait | 错误类型自动转换（配合 `?`） |

---

## 🎓 思考题

为什么 Rust 要求 `Error` trait 同时实现 `Debug` 和 `Display`？

**答案：** 因为错误信息有两种受众：
- `Display` — 给终端用户看，应该简洁友好
- `Debug` — 给开发者调试用，应该包含技术细节

```rust
// 给用户看
println!("错误: {}", error);  // Display

// 给开发者调试
println!("调试信息: {:?}", error);  // Debug
```

---

*🎉 第 100 课完成！感谢一路相伴，我们已经系统学习了 Rust 语言的核心内容。接下来会继续深入更多高级主题！*
