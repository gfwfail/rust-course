# 第34课：测试 (Testing)

> 日期：2026-02-17  
> 主题：单元测试、集成测试、文档测试

---

## 为什么要学测试？

Rust 的测试系统是内置的，不需要额外安装框架。写测试在 Rust 里非常简单，而且编译器会帮你确保测试和代码保持同步。

---

## 基础：单元测试

最简单的测试就是在代码文件里加一个测试模块：

```rust
// src/lib.rs

pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

pub fn divide(a: i32, b: i32) -> Option<i32> {
    if b == 0 {
        None
    } else {
        Some(a / b)
    }
}

// 测试模块 —— 只在 cargo test 时编译
#[cfg(test)]
mod tests {
    use super::*;  // 导入父模块的所有内容

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn test_add_negative() {
        assert_eq!(add(-1, 1), 0);
    }

    #[test]
    fn test_divide() {
        assert_eq!(divide(10, 2), Some(5));
    }

    #[test]
    fn test_divide_by_zero() {
        assert_eq!(divide(10, 0), None);
    }
}
```

运行测试：
```bash
cargo test
```

---

## 常用断言宏

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn assertions_demo() {
        // 相等判断
        assert_eq!(1 + 1, 2);
        
        // 不相等判断
        assert_ne!(1 + 1, 3);
        
        // 布尔判断
        assert!(true);
        assert!(!false);
        
        // 带自定义消息
        assert_eq!(
            1 + 1, 
            2, 
            "数学崩了：1 + 1 应该等于 2"
        );
    }
}
```

---

## 测试 panic

有时候你需要验证代码在某些情况下会 panic：

```rust
pub fn divide_strict(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("不能除以零！");
    }
    a / b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn test_divide_by_zero_panics() {
        divide_strict(10, 0);  // 应该 panic
    }

    // 更精确：检查 panic 消息
    #[test]
    #[should_panic(expected = "不能除以零")]
    fn test_panic_message() {
        divide_strict(10, 0);
    }
}
```

---

## 使用 Result 的测试

测试函数也可以返回 Result，失败时返回 Err：

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_with_result() -> Result<(), String> {
        let value = "42".parse::<i32>()
            .map_err(|e| e.to_string())?;
        
        if value == 42 {
            Ok(())
        } else {
            Err(format!("期望 42，得到 {}", value))
        }
    }
}
```

---

## 运行特定测试

```bash
# 运行所有测试
cargo test

# 运行名字包含 "divide" 的测试
cargo test divide

# 运行单个测试
cargo test test_add

# 显示 println! 输出（默认被捕获）
cargo test -- --nocapture

# 单线程运行（测试有依赖时）
cargo test -- --test-threads=1
```

---

## 忽略测试

```rust
#[test]
#[ignore]  // 默认跳过
fn expensive_test() {
    // 运行很慢的测试
    std::thread::sleep(std::time::Duration::from_secs(10));
}

// 只运行被忽略的测试
// cargo test -- --ignored
```

---

## 集成测试

集成测试放在 `tests/` 目录，测试你的库的公开 API：

```
my_project/
├── src/
│   └── lib.rs
├── tests/           ← 集成测试目录
│   ├── integration_test.rs
│   └── api_test.rs
└── Cargo.toml
```

```rust
// tests/integration_test.rs
// 注意：这里把你的 crate 当作外部库来用

use my_project::{add, divide};

#[test]
fn test_add_from_outside() {
    assert_eq!(add(10, 20), 30);
}

#[test]
fn test_divide_from_outside() {
    assert_eq!(divide(100, 10), Some(10));
}
```

集成测试只能访问 `pub` 的内容，这是故意的——它模拟真实用户使用你的库。

---

## 测试私有函数

单元测试可以测试私有函数（因为在同一个模块）：

```rust
// src/lib.rs

fn internal_helper(x: i32) -> i32 {
    x * 2
}

pub fn public_fn(x: i32) -> i32 {
    internal_helper(x) + 1
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_internal() {
        // ✅ 可以访问私有函数
        assert_eq!(internal_helper(5), 10);
    }
}
```

---

## 测试辅助模块

当多个测试需要共享代码时：

```
tests/
├── common/
│   └── mod.rs    ← 共享的测试工具
├── test_a.rs
└── test_b.rs
```

```rust
// tests/common/mod.rs
pub fn setup() -> TestContext {
    // 初始化测试环境
    TestContext::new()
}

pub struct TestContext {
    pub db_url: String,
}

impl TestContext {
    pub fn new() -> Self {
        Self {
            db_url: "sqlite::memory:".to_string(),
        }
    }
}
```

```rust
// tests/test_a.rs
mod common;

#[test]
fn test_with_setup() {
    let ctx = common::setup();
    // 使用 ctx...
}
```

---

## Doc Tests（文档测试）

Rust 可以自动运行文档中的代码示例：

```rust
/// 将两个数字相加
/// 
/// # Examples
/// 
/// ```
/// use my_project::add;
/// 
/// let result = add(2, 3);
/// assert_eq!(result, 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

运行 `cargo test` 会自动测试文档中的代码块！

隐藏某些行（用 `#`）：
```rust
/// # Examples
/// 
/// ```
/// # use my_project::divide;
/// # fn main() -> Option<()> {
/// let result = divide(10, 2)?;
/// assert_eq!(result, 5);
/// # Some(())
/// # }
/// ```
```

---

## 测试最佳实践

### 1. 测试命名要清晰

```rust
// ❌ 不好
#[test]
fn test1() { }

// ✅ 好
#[test]
fn add_returns_sum_of_two_positive_numbers() { }

#[test]
fn divide_returns_none_when_divisor_is_zero() { }
```

### 2. 一个测试只测一件事

```rust
// ❌ 不好：测试太多东西
#[test]
fn test_everything() {
    assert_eq!(add(1, 2), 3);
    assert_eq!(divide(10, 2), Some(5));
    assert_eq!(divide(10, 0), None);
}

// ✅ 好：分开测试
#[test]
fn add_works() { assert_eq!(add(1, 2), 3); }

#[test]
fn divide_works() { assert_eq!(divide(10, 2), Some(5)); }

#[test]
fn divide_by_zero_returns_none() { assert_eq!(divide(10, 0), None); }
```

### 3. 测试边界情况

```rust
#[test]
fn test_edge_cases() {
    assert_eq!(add(0, 0), 0);
    assert_eq!(add(i32::MAX, 0), i32::MAX);
    assert_eq!(add(-1, 1), 0);
}
```

---

## 小结

| 概念 | 说明 |
|------|------|
| `#[test]` | 标记测试函数 |
| `#[cfg(test)]` | 只在测试时编译 |
| `assert!`, `assert_eq!`, `assert_ne!` | 断言宏 |
| `#[should_panic]` | 期望 panic |
| `#[ignore]` | 跳过测试 |
| `tests/` 目录 | 集成测试 |
| `///` 文档注释 | 文档测试 |

---

## 下一课预告

下节课我们将学习 **宏 (Macros)**——Rust 的元编程能力！
