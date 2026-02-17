# 第 40 课：Cargo Features - 条件编译与功能开关

> 日期：2026-02-17  
> 主题：Features、条件编译、可选依赖

---

## 什么是 Features？

Features 是 Cargo 的条件编译机制，用于：
- 根据不同场景启用/禁用某些功能
- 减少编译时间和二进制体积
- 提供可选的依赖

类似于 PHP 的 require-dev、JS 的构建时环境变量、C 的 `#ifdef`，但更加类型安全！

---

## 基础用法

### 1. 在 Cargo.toml 中定义 Features

```toml
[package]
name = "my_app"
version = "0.1.0"

[features]
# 默认启用的 features
default = ["json"]

# 定义各个 feature
json = ["dep:serde_json"]
xml = ["dep:quick-xml"]
full = ["json", "xml", "async"]  # 组合多个 features
async = ["dep:tokio"]

[dependencies]
serde = "1.0"
serde_json = { version = "1.0", optional = true }
quick-xml = { version = "0.31", optional = true }
tokio = { version = "1", features = ["full"], optional = true }
```

### 2. 在代码中使用条件编译

```rust
// 只在启用 json feature 时编译
#[cfg(feature = "json")]
pub mod json_support {
    use serde_json::Value;
    
    pub fn parse_json(s: &str) -> Result<Value, serde_json::Error> {
        serde_json::from_str(s)
    }
}

// 只在启用 xml feature 时编译
#[cfg(feature = "xml")]
pub mod xml_support {
    pub fn parse_xml(s: &str) -> String {
        format!("Parsed: {}", s)
    }
}

fn main() {
    #[cfg(feature = "json")]
    {
        println!("JSON support enabled!");
        let data = json_support::parse_json(r#"{"name": "rust"}"#);
        println!("{:?}", data);
    }

    #[cfg(feature = "xml")]
    println!("XML support enabled!");

    #[cfg(not(feature = "json"))]
    println!("JSON support disabled");
}
```

---

## 编译命令

```bash
# 使用默认 features
cargo build

# 禁用默认 features
cargo build --no-default-features

# 启用指定 features
cargo build --features "json xml"

# 禁用默认 + 启用指定
cargo build --no-default-features --features "xml"

# 启用所有 features（开发时有用）
cargo build --all-features
```

---

## 实战场景

### 场景 1：开发 vs 生产配置

```toml
[features]
default = []
dev-tools = ["dep:tracing-subscriber", "dep:console"]
```

```rust
#[cfg(feature = "dev-tools")]
fn setup_logging() {
    tracing_subscriber::fmt::init();
    println!("Dev logging enabled");
}

#[cfg(not(feature = "dev-tools"))]
fn setup_logging() {
    // 生产环境用更轻量的日志
}
```

### 场景 2：多数据库支持

```toml
[features]
default = ["postgres"]
postgres = ["dep:sqlx/postgres"]
mysql = ["dep:sqlx/mysql"]
sqlite = ["dep:sqlx/sqlite"]
```

```rust
#[cfg(feature = "postgres")]
pub type DbPool = sqlx::PgPool;

#[cfg(feature = "mysql")]
pub type DbPool = sqlx::MySqlPool;

#[cfg(feature = "sqlite")]
pub type DbPool = sqlx::SqlitePool;
```

### 场景 3：同步 vs 异步 API

```toml
[features]
default = ["blocking"]
blocking = ["dep:reqwest/blocking"]
async = ["dep:reqwest", "dep:tokio"]
```

```rust
#[cfg(feature = "blocking")]
pub fn fetch(url: &str) -> String {
    reqwest::blocking::get(url)
        .unwrap()
        .text()
        .unwrap()
}

#[cfg(feature = "async")]
pub async fn fetch(url: &str) -> String {
    reqwest::get(url)
        .await
        .unwrap()
        .text()
        .await
        .unwrap()
}
```

---

## 重要注意事项

### 1. Feature 必须是 Additive（加法原则）

Features 只能增加功能，不能改变行为：

```rust
// ❌ 错误：features 改变行为
#[cfg(feature = "fast")]
fn calculate() -> i32 { 1 }

#[cfg(not(feature = "fast"))]
fn calculate() -> i32 { 2 }  // 不同的结果！

// ✅ 正确：features 只增加功能
fn calculate() -> i32 { 1 }

#[cfg(feature = "fast")]
fn calculate_fast() -> i32 { 
    // 更快的算法，但结果一样
    1 
}
```

### 2. 检查 Feature 组合

```rust
// 确保不会同时启用冲突的 features
#[cfg(all(feature = "postgres", feature = "mysql"))]
compile_error!("Cannot enable both postgres and mysql");

// 确保至少启用一个数据库
#[cfg(not(any(feature = "postgres", feature = "mysql", feature = "sqlite")))]
compile_error!("Must enable at least one database feature");
```

### 3. cfg_attr 条件属性

```rust
// 根据 feature 应用不同的 derive
#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
#[derive(Debug, Clone)]
pub struct User {
    pub name: String,
    pub age: u32,
}
```

---

## 高级技巧

### 1. Feature 依赖传递

```toml
[features]
tls = ["dep:native-tls"]
tls-rustls = ["dep:rustls"]
default-tls = ["tls"]

[dependencies]
native-tls = { version = "0.2", optional = true }
rustls = { version = "0.21", optional = true }
```

### 2. 依赖的 Feature 控制

```toml
[dependencies]
# 启用依赖的特定 features
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }

# 根据自己的 feature 启用依赖的 feature
serde = { version = "1.0", features = ["derive"] }
sqlx = { version = "0.7", default-features = false }

[features]
postgres = ["sqlx/postgres", "sqlx/runtime-tokio"]
mysql = ["sqlx/mysql", "sqlx/runtime-tokio"]
```

### 3. 运行时 Feature 检测

```rust
// 编译时生成 feature 信息
pub const ENABLED_FEATURES: &[&str] = &[
    #[cfg(feature = "json")]
    "json",
    #[cfg(feature = "xml")]
    "xml",
    #[cfg(feature = "async")]
    "async",
];

fn print_features() {
    println!("Enabled features: {:?}", ENABLED_FEATURES);
}
```

---

## 对比其他语言

| 语言 | 条件编译方式 |
|------|-------------|
| PHP | 运行时 `if (extension_loaded(...))` |
| JS | 构建工具 + 环境变量 |
| C | `#ifdef` 预处理器 |
| **Rust** | `#[cfg]` + Cargo features |

Rust 的优势：
- ✅ 编译时检查，无运行时开销
- ✅ 类型安全
- ✅ 依赖自动管理
- ✅ IDE 支持良好

---

## 要点总结

1. **Features 定义** 在 `Cargo.toml` 的 `[features]` 部分
2. **条件编译** 用 `#[cfg(feature = "xxx")]`
3. **命令行控制** 用 `--features` / `--no-default-features`
4. **Additive 原则**：features 只加功能，不改行为
5. **可选依赖** 用 `optional = true` + `dep:xxx`

---

## 下一课预告

Build Scripts (build.rs) - 编译时代码生成
