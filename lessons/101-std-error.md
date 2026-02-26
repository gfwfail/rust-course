# 第 101 课：std::error — Error trait 与错误链

今天深入标准库的 **Error trait**——Rust 错误处理的基石。

---

## 🎯 为什么要讲 Error trait？

之前用过 `anyhow` 和 `thiserror`，但它们都基于标准库的 `std::error::Error`。理解底层，才能用好上层。

```rust
// 标准库定义（简化版）
pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

核心三点：
1. 必须实现 `Debug`（给开发者看）
2. 必须实现 `Display`（给用户看）
3. 可选实现 `source()`（错误链）

---

## 📦 手写一个标准错误类型

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct MyError {
    message: String,
    source: Option<Box<dyn Error + Send + Sync>>,
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.message)
    }
}

impl Error for MyError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        self.source.as_ref()
            .map(|e| e.as_ref() as &(dyn Error + 'static))
    }
}

// 使用
fn might_fail() -> Result<(), MyError> {
    Err(MyError {
        message: "操作失败".into(),
        source: None,
    })
}
```

---

## 🔗 source() — 错误链的魔法

错误链让你追溯"错误的错误"：

```rust
use std::fs::File;
use std::io;

#[derive(Debug)]
struct ConfigError {
    path: String,
    source: io::Error,
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "无法加载配置文件: {}", self.path)
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        Some(&self.source)  // 暴露底层 io::Error
    }
}

fn load_config(path: &str) -> Result<String, ConfigError> {
    std::fs::read_to_string(path).map_err(|e| ConfigError {
        path: path.to_string(),
        source: e,
    })
}

// 遍历错误链
fn print_error_chain(err: &dyn Error) {
    println!("Error: {}", err);
    let mut current = err.source();
    while let Some(cause) = current {
        println!("Caused by: {}", cause);
        current = cause.source();
    }
}
```

输出：
```
Error: 无法加载配置文件: /etc/app.conf
Caused by: No such file or directory (os error 2)
```

---

## 📦 Box<dyn Error> — 类型擦除的错误

当你不关心具体错误类型时：

```rust
use std::error::Error;

// 返回任意错误
fn do_something() -> Result<(), Box<dyn Error>> {
    let _file = std::fs::File::open("不存在.txt")?;
    let _num: i32 = "abc".parse()?;  // 不同类型的错误！
    Ok(())
}

// 线程安全版本
fn do_something_threadsafe() -> Result<(), Box<dyn Error + Send + Sync>> {
    // 可以跨线程传递错误
    Ok(())
}
```

**对比 PHP/Laravel：**
```php
// PHP 只能 catch 一个基类
try {
    // ...
} catch (Exception $e) {
    // 所有异常都变成 Exception
}
```

Rust 的 `Box<dyn Error>` 保留了错误链信息，比 PHP 更强大。

---

## 🔍 downcast — 恢复具体类型

有时需要知道具体是什么错误：

```rust
use std::error::Error;
use std::io;

fn handle_error(err: Box<dyn Error>) {
    // 尝试转换为具体类型
    if let Some(io_err) = err.downcast_ref::<io::Error>() {
        match io_err.kind() {
            io::ErrorKind::NotFound => println!("文件不存在"),
            io::ErrorKind::PermissionDenied => println!("权限不足"),
            _ => println!("IO 错误: {}", io_err),
        }
    } else {
        println!("其他错误: {}", err);
    }
}

// downcast 系列方法
// downcast_ref::<T>() -> Option<&T>      // 借用
// downcast_mut::<T>() -> Option<&mut T>  // 可变借用
// downcast::<T>() -> Result<Box<T>, Self> // 消耗所有权
```

---

## 🆕 Rust 1.65+：provide() 和 request()

新版 Rust 引入了更灵活的错误上下文（还在 nightly）：

```rust
#![feature(error_generic_member_access)]

use std::error::Error;
use std::backtrace::Backtrace;

#[derive(Debug)]
struct MyError {
    message: String,
    backtrace: Backtrace,
}

impl Error for MyError {
    fn provide<'a>(&'a self, request: &mut std::error::Request<'a>) {
        request.provide_ref::<Backtrace>(&self.backtrace);
    }
}

// 获取 backtrace（如果有）
fn get_backtrace(err: &dyn Error) -> Option<&Backtrace> {
    err.request_ref::<Backtrace>()
}
```

---

## 💡 实战对比：手写 vs thiserror

**手写：**
```rust
#[derive(Debug)]
struct ParseConfigError {
    line: usize,
    source: std::num::ParseIntError,
}

impl fmt::Display for ParseConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "配置解析失败，第 {} 行", self.line)
    }
}

impl Error for ParseConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        Some(&self.source)
    }
}
```

**用 thiserror：**
```rust
use thiserror::Error;

#[derive(Debug, Error)]
#[error("配置解析失败，第 {line} 行")]
struct ParseConfigError {
    line: usize,
    #[source]
    source: std::num::ParseIntError,
}
```

thiserror 自动实现 `Display` 和 `Error`，省了 20 行代码！

---

## 🧠 核心要点

| 特性 | 说明 |
|------|------|
| `Debug + Display` | Error trait 的基础要求 |
| `source()` | 返回导致当前错误的底层错误 |
| `Box<dyn Error>` | 类型擦除，接受任何错误 |
| `downcast_ref::<T>()` | 恢复具体错误类型 |
| `+ Send + Sync` | 线程安全的错误 |

---

## 📝 小练习

写一个 `HttpError`，包含状态码和可选的底层 `io::Error`，实现完整的 Error trait。

---

*日期：2026-02-27*
