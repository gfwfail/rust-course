# 第 101 课：std::error — 错误处理的艺术

> 日期：2026-02-26  
> 主题：std::error 模块与 Error trait

---

## 为什么需要 std::error？

Rust 的错误处理基于 `Result<T, E>`，但标准库提供了 `std::error::Error` trait 来统一错误类型的行为，让不同类型的错误可以互操作。

```rust
// Error trait 的定义（简化版）
pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

三个关键点：
1. 必须实现 `Debug`（调试输出）
2. 必须实现 `Display`（用户友好的输出）
3. 可选实现 `source()`（错误链追踪）

---

## 自定义错误类型

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct MyError {
    message: String,
    source: Option<Box<dyn Error + 'static>>,
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.message)
    }
}

impl Error for MyError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        self.source.as_ref().map(|e| e.as_ref())
    }
}
```

---

## 错误链（Error Chain）

`source()` 方法让你可以追踪错误的根因：

```rust
fn print_error_chain(err: &dyn Error) {
    println!("Error: {}", err);
    
    let mut source = err.source();
    while let Some(cause) = source {
        println!("Caused by: {}", cause);
        source = cause.source();
    }
}

// 输出示例：
// Error: 无法读取配置文件
// Caused by: 文件不存在
// Caused by: No such file or directory (os error 2)
```

---

## Box<dyn Error> 作为通用错误类型

```rust
use std::error::Error;

// 可以返回任何实现了 Error 的类型
fn do_something() -> Result<(), Box<dyn Error>> {
    let _file = std::fs::read_to_string("config.txt")?;
    let _num: i32 = "not a number".parse()?;
    Ok(())
}

fn main() {
    if let Err(e) = do_something() {
        print_error_chain(e.as_ref());
    }
}
```

**优点**：简单、灵活  
**缺点**：堆分配、无法精确匹配错误类型

---

## 对比 PHP/Laravel

```php
// PHP 的异常链
try {
    throw new Exception("底层错误");
} catch (Exception $e) {
    throw new Exception("上层错误", 0, $e);
}

// 获取原因
$previous = $exception->getPrevious();
```

Rust 用 `source()` 实现类似功能，但：
- 不用异常机制（无运行时开销）
- 编译期强制处理
- 错误链是可选的

---

## 标准库中实现 Error 的类型

```rust
// 常见错误类型
std::io::Error          // IO 错误
std::num::ParseIntError // 数字解析错误
std::str::Utf8Error     // UTF-8 解码错误
std::fmt::Error         // 格式化错误
std::env::VarError      // 环境变量错误

// 检查类型
fn handle_error(err: &dyn Error) {
    if let Some(io_err) = err.downcast_ref::<std::io::Error>() {
        println!("IO error: {:?}", io_err.kind());
    }
}
```

---

## downcast：向下转型

```rust
use std::error::Error;

fn try_recover(err: Box<dyn Error>) {
    // 尝试转换为具体类型
    if let Some(io_err) = err.downcast_ref::<std::io::Error>() {
        match io_err.kind() {
            std::io::ErrorKind::NotFound => {
                println!("文件不存在，创建默认配置...");
            }
            _ => println!("其他 IO 错误"),
        }
    }
}
```

---

## 实战：优雅的错误处理模式

```rust
use std::error::Error;
use std::fmt;

// 1. 定义错误枚举
#[derive(Debug)]
enum AppError {
    Config(String),
    Database(String),
    Network { url: String, source: Box<dyn Error> },
}

// 2. 实现 Display
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Config(msg) => write!(f, "配置错误: {}", msg),
            AppError::Database(msg) => write!(f, "数据库错误: {}", msg),
            AppError::Network { url, .. } => {
                write!(f, "网络错误: 无法连接 {}", url)
            }
        }
    }
}

// 3. 实现 Error
impl Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            AppError::Network { source, .. } => Some(source.as_ref()),
            _ => None,
        }
    }
}

// 4. 实现 From 转换
impl From<std::io::Error> for AppError {
    fn from(err: std::io::Error) -> Self {
        AppError::Config(err.to_string())
    }
}
```

---

## 小结

| 概念 | 说明 |
|------|------|
| `Error` trait | 统一错误接口 |
| `source()` | 错误链追踪 |
| `Box<dyn Error>` | 通用错误类型 |
| `downcast_ref()` | 向下转型匹配 |

**经验法则**：
- 库代码：定义自己的错误枚举
- 应用代码：用 `Box<dyn Error>` 或 `anyhow`
- 总是实现 `Display` 给用户看

---

## 下节预告

下节课：`std::convert` — 类型转换的系统化方法

---

*笔记整理：性奴001*
