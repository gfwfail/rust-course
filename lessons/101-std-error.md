# 第 101 课：std::error — Error trait 与自定义错误

> 日期：2026-03-01

---

## 📚 为什么要学 std::error？

之前我们用过 `Result<T, E>`，也用过 `anyhow` 和 `thiserror`。但你知道它们底层都依赖什么吗？

答案是：`std::error::Error` trait。

这是 Rust 错误处理的**基石**。理解它，你才能：
- 自己写出专业的错误类型
- 理解第三方库的错误设计
- 正确地链式传播错误

---

## 🔍 Error trait 的定义

```rust
pub trait Error: Debug + Display {
    // 返回导致这个错误的"源头"错误
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None  // 默认没有源头
    }
}
```

就这么简单！Error trait 只要求：
1. 实现 `Debug`（用于 `{:?}` 打印）
2. 实现 `Display`（用于 `{}` 打印）
3. 可选实现 `source()`（错误链）

---

## 💡 手写一个自定义错误

```rust
use std::error::Error;
use std::fmt;

// 1. 定义错误类型
#[derive(Debug)]
struct MyError {
    message: String,
    source: Option<Box<dyn Error + 'static>>,
}

// 2. 实现 Display
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

// 4. 便捷构造方法
impl MyError {
    fn new(msg: impl Into<String>) -> Self {
        Self { message: msg.into(), source: None }
    }
    
    fn with_source(msg: impl Into<String>, source: impl Error + 'static) -> Self {
        Self { 
            message: msg.into(), 
            source: Some(Box::new(source)) 
        }
    }
}
```

---

## 🔗 错误链的威力

`source()` 方法让你可以追溯错误的"根本原因"：

```rust
use std::fs::File;
use std::io;

fn read_config() -> Result<String, MyError> {
    let file = File::open("config.toml")
        .map_err(|e| MyError::with_source("无法读取配置文件", e))?;
    // ...
    Ok(String::new())
}

fn main() {
    if let Err(e) = read_config() {
        println!("错误: {}", e);
        
        // 遍历错误链
        let mut source = e.source();
        while let Some(s) = source {
            println!("  原因: {}", s);
            source = s.source();
        }
    }
}

// 输出:
// 错误: 无法读取配置文件
//   原因: No such file or directory (os error 2)
```

---

## 🆚 对比 PHP/Laravel

```php
// PHP: 异常链
try {
    throw new Exception("原始错误");
} catch (Exception $e) {
    throw new RuntimeException("包装错误", 0, $e);
}

// 获取原因
$e->getPrevious();
```

Rust 的 `source()` 就相当于 PHP 的 `getPrevious()`。

---

## ⚡ 实用方法

### 1. 快速打印错误链

```rust
fn print_error_chain(err: &dyn Error) {
    eprintln!("Error: {}", err);
    let mut source = err.source();
    while let Some(s) = source {
        eprintln!("  Caused by: {}", s);
        source = s.source();
    }
}
```

### 2. 用 std 的方法遍历错误链（nightly）

```rust
#![feature(error_iter)]

for cause in err.sources() {
    eprintln!("{}", cause);
}
```

---

## 🎯 实战：多种错误类型

```rust
use std::error::Error;
use std::fmt;
use std::num::ParseIntError;
use std::io;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
    Custom(String),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "IO 错误: {}", e),
            AppError::Parse(e) => write!(f, "解析错误: {}", e),
            AppError::Custom(msg) => write!(f, "{}", msg),
        }
    }
}

impl Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            AppError::Io(e) => Some(e),
            AppError::Parse(e) => Some(e),
            AppError::Custom(_) => None,
        }
    }
}

// From trait 让 ? 操作符自动转换
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
```

现在你可以这样用：

```rust
fn process() -> Result<i32, AppError> {
    let content = std::fs::read_to_string("number.txt")?;  // io::Error → AppError
    let num: i32 = content.trim().parse()?;  // ParseIntError → AppError
    Ok(num * 2)
}
```

---

## 🧠 要点总结

| 概念 | 说明 |
|------|------|
| `Error` trait | 需要 `Debug` + `Display` |
| `source()` | 返回导致这个错误的底层错误 |
| 错误链 | 通过 `source()` 链式追溯 |
| `From` trait | 配合 `?` 自动转换错误类型 |

---

## 💭 思考题

为什么 `thiserror` 和 `anyhow` 这么流行？

因为手写 `Display`、`Error`、`From` 太繁琐了！
- `thiserror` 用宏帮你自动 derive
- `anyhow` 直接 `Box<dyn Error>` 一把梭

但理解底层原理，你才能在需要时写出精准的错误类型。

---

📝 **下节预告**: `std::panic` — panic 处理与 catch_unwind
