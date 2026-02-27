# 第 101 课：std::error — Error trait 错误处理的核心

> 日期：2026-02-27  
> 主题：Error trait、错误链、类型擦除与 downcast

---

## 🎯 为什么要学 Error trait？

我们用了很多 `Result<T, E>`，但你有没有想过：
- 为什么 `?` 可以自动转换不同类型的错误？
- 为什么可以用 `Box<dyn Error>` 装任何错误？
- 错误链是怎么工作的？

答案就在 `Error` trait。

---

## 📖 Error trait 定义

```rust
// std::error::Error（简化版）
pub trait Error: Debug + Display {
    // 返回导致这个错误的底层错误（如果有）
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

核心要点：
1. **必须实现 `Debug` 和 `Display`** — 错误必须能打印
2. **`source()` 方法** — 用于错误链追溯

---

## 🔧 自定义错误类型

### 最简单的自定义错误

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct MyError {
    message: String,
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "MyError: {}", self.message)
    }
}

impl Error for MyError {} // source() 默认返回 None

fn might_fail(flag: bool) -> Result<(), MyError> {
    if flag {
        Ok(())
    } else {
        Err(MyError {
            message: "Something went wrong".to_string(),
        })
    }
}

fn main() {
    match might_fail(false) {
        Ok(_) => println!("Success!"),
        Err(e) => {
            println!("{}", e);        // Display
            println!("{:?}", e);      // Debug
        }
    }
}
```

---

## ⛓️ 错误链 (Error Chain)

当一个错误是由另一个错误引起时，用 `source()` 建立链接：

```rust
use std::error::Error;
use std::fmt;
use std::io;
use std::fs;

#[derive(Debug)]
struct ConfigError {
    path: String,
    source: io::Error,  // 保存底层错误
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Failed to load config from '{}'", self.path)
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        Some(&self.source)  // 返回底层 io::Error
    }
}

fn load_config(path: &str) -> Result<String, ConfigError> {
    fs::read_to_string(path).map_err(|e| ConfigError {
        path: path.to_string(),
        source: e,
    })
}

fn main() {
    match load_config("/nonexistent/config.toml") {
        Ok(content) => println!("{}", content),
        Err(e) => {
            // 遍历错误链
            println!("Error: {}", e);
            
            let mut source = e.source();
            while let Some(cause) = source {
                println!("Caused by: {}", cause);
                source = cause.source();
            }
        }
    }
}
```

输出：
```
Error: Failed to load config from '/nonexistent/config.toml'
Caused by: No such file or directory (os error 2)
```

---

## 📦 Box<dyn Error> — 类型擦除

当函数可能返回多种错误类型时：

```rust
use std::error::Error;
use std::fs;
use std::num::ParseIntError;

// Box<dyn Error> 可以装任何实现了 Error 的类型
fn read_number(path: &str) -> Result<i32, Box<dyn Error>> {
    let content = fs::read_to_string(path)?;  // io::Error
    let num: i32 = content.trim().parse()?;   // ParseIntError
    Ok(num)
}

fn main() {
    match read_number("number.txt") {
        Ok(n) => println!("Number: {}", n),
        Err(e) => println!("Error: {}", e),
    }
}
```

为什么 `?` 能自动转换？因为标准库实现了：
```rust
impl<E: Error + 'static> From<E> for Box<dyn Error>
```

---

## 🔄 downcast — 从 dyn Error 还原具体类型

有时候你需要判断具体是什么错误：

```rust
use std::error::Error;
use std::io;

fn handle_error(e: Box<dyn Error>) {
    // 尝试 downcast 到具体类型
    if let Some(io_err) = e.downcast_ref::<io::Error>() {
        match io_err.kind() {
            io::ErrorKind::NotFound => {
                println!("File not found!");
            }
            io::ErrorKind::PermissionDenied => {
                println!("Permission denied!");
            }
            _ => println!("IO error: {}", io_err),
        }
    } else {
        println!("Unknown error: {}", e);
    }
}
```

downcast 方法家族：
- `downcast_ref::<T>()` → `Option<&T>`
- `downcast_mut::<T>()` → `Option<&mut T>`
- `downcast::<T>()` → `Result<Box<T>, Box<dyn Error>>`

---

## 🆚 对比：PHP/Laravel 的异常

| 概念 | PHP | Rust |
|------|-----|------|
| 错误类型 | Exception 类 | 实现 Error trait |
| 错误链 | `$e->getPrevious()` | `e.source()` |
| 捕获所有 | `catch (Throwable $e)` | `Box<dyn Error>` |
| 类型判断 | `$e instanceof` | `downcast_ref::<T>()` |
| 打印 | `$e->getMessage()` | `Display` trait |

```php
// PHP
try {
    throw new Exception("inner");
} catch (Exception $e) {
    throw new Exception("outer", 0, $e);  // 错误链
}

// Rust 等价
Err(OuterError { source: inner_error })
```

---

## 💡 Rust 1.81+ 的改进

从 Rust 1.81 开始，`Error` trait 在 `core` 里也可用了（之前只在 `std`）。这意味着 `no_std` 环境也能用标准的错误处理。

---

## 📝 最佳实践总结

1. **自定义错误类型时**：实现 `Debug`、`Display`、`Error`
2. **保留错误上下文**：用 `source()` 建立错误链
3. **快速开发**：用 `Box<dyn Error>` 或 `anyhow::Error`
4. **库代码**：定义具体错误枚举 + `thiserror`
5. **错误处理**：优先模式匹配，必要时 `downcast`

---

## 🏋️ 练习

写一个解析 JSON 配置的函数，要求：
1. 自定义 `ConfigError` 枚举（FileNotFound、ParseError、ValidationError）
2. 每个变体包含导致它的底层错误
3. 实现完整的 `Error` trait 包括 `source()`

---

*性奴001 | 2026-02-27*
