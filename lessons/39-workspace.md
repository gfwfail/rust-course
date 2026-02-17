# 第 39 课：Cargo Workspace 工作空间

> 大型项目的模块化组织方式

---

## 为什么需要 Workspace？

当项目变大，你会遇到这些问题：
- 代码越来越多，一个 crate 太臃肿
- 想把核心逻辑、API 层、CLI 工具分开
- 多个 crate 之间需要共享依赖版本
- 想一次 `cargo build` 编译所有东西

**Workspace 就是解决方案！**

类比 PHP/Laravel：就像 Composer 的 monorepo，或者用 path repository 引用本地包。

---

## 基本结构

```
my-project/
├── Cargo.toml          # workspace 根配置
├── crates/
│   ├── core/           # 核心逻辑
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── api/            # Web API 层
│   │   ├── Cargo.toml
│   │   └── src/main.rs
│   └── cli/            # 命令行工具
│       ├── Cargo.toml
│       └── src/main.rs
└── target/             # 共享的编译输出
```

---

## 根 Cargo.toml

```toml
[workspace]
resolver = "2"
members = [
    "crates/core",
    "crates/api",
    "crates/cli",
]

# 统一管理依赖版本
[workspace.dependencies]
tokio = { version = "1.36", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
anyhow = "1.0"
tracing = "0.1"

# 统一 package 元数据
[workspace.package]
version = "0.1.0"
edition = "2021"
authors = ["Your Name"]
license = "MIT"
```

---

## 子 crate 的 Cargo.toml

```toml
[package]
name = "my-core"
# 继承 workspace 的 package 配置
version.workspace = true
edition.workspace = true
authors.workspace = true

[dependencies]
# 从 workspace 继承依赖
tokio.workspace = true
serde.workspace = true

# 本地依赖（引用其他 crate）
# 不需要写 path，因为在同一个 workspace

[lib]
name = "my_core"
```

API crate 引用 core：

```toml
[package]
name = "my-api"
version.workspace = true
edition.workspace = true

[dependencies]
tokio.workspace = true
my-core = { path = "../core" }
```

---

## 常用命令

```bash
# 构建所有 crate
cargo build

# 只构建特定 crate
cargo build -p my-api

# 运行特定 binary
cargo run -p my-cli

# 测试所有 crate
cargo test

# 测试特定 crate
cargo test -p my-core

# 检查所有 crate
cargo check --all
```

---

## 实战技巧

### 1. 依赖去重

Workspace 统一管理依赖版本，避免：
- crate A 用 `tokio 1.35`
- crate B 用 `tokio 1.36`
- 编译两份 tokio！

### 2. 典型分层架构

```
crates/
├── domain/       # 领域模型、业务逻辑（纯 Rust，无 IO）
├── infra/        # 数据库、外部服务适配器
├── api/          # HTTP 层（Axum handlers）
└── cli/          # CLI 工具
```

### 3. Feature Flags 跨 crate

```toml
# workspace Cargo.toml
[workspace.dependencies]
my-core = { path = "crates/core", default-features = false }

# 在 api/Cargo.toml
[dependencies]
my-core = { workspace = true, features = ["postgres"] }
```

---

## 注意事项

### 1. 路径依赖不需要 version

```toml
# ✅ 正确
my-core = { path = "../core" }

# ❌ 错误（除非要发布到 crates.io）
my-core = { path = "../core", version = "0.1" }
```

### 2. 共享 target 目录

- 好处：编译一次，所有 crate 共用
- 坏处：单个 crate 的 `cargo clean` 会清掉所有

### 3. 发布顺序

如果要发到 crates.io，必须按依赖顺序发布：

```bash
cargo publish -p my-core
cargo publish -p my-api  # 依赖 my-core
```

---

## 完整示例

### 步骤 1：创建目录结构

```bash
mkdir rust-math && cd rust-math
mkdir -p crates/core crates/cli
```

### 步骤 2：根 Cargo.toml

```toml
[workspace]
resolver = "2"
members = ["crates/core", "crates/cli"]

[workspace.package]
version = "0.1.0"
edition = "2021"

[workspace.dependencies]
clap = { version = "4.5", features = ["derive"] }
```

### 步骤 3：core/Cargo.toml

```toml
[package]
name = "math-core"
version.workspace = true
edition.workspace = true

[lib]
```

### 步骤 4：core/src/lib.rs

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

pub fn subtract(a: i32, b: i32) -> i32 {
    a - b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn test_subtract() {
        assert_eq!(subtract(5, 3), 2);
    }
}
```

### 步骤 5：cli/Cargo.toml

```toml
[package]
name = "math-cli"
version.workspace = true
edition.workspace = true

[dependencies]
clap.workspace = true
math-core = { path = "../core" }

[[bin]]
name = "calc"
path = "src/main.rs"
```

### 步骤 6：cli/src/main.rs

```rust
use clap::{Parser, Subcommand};
use math_core::{add, subtract};

#[derive(Parser)]
#[command(name = "calc")]
#[command(about = "A simple calculator")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Add two numbers
    Add { a: i32, b: i32 },
    /// Subtract two numbers
    Sub { a: i32, b: i32 },
}

fn main() {
    let cli = Cli::parse();

    match cli.command {
        Commands::Add { a, b } => {
            println!("{} + {} = {}", a, b, add(a, b));
        }
        Commands::Sub { a, b } => {
            println!("{} - {} = {}", a, b, subtract(a, b));
        }
    }
}
```

### 运行测试

```bash
# 测试 core
cargo test -p math-core

# 运行 CLI
cargo run -p math-cli -- add 10 20
# 输出: 10 + 20 = 30

cargo run -p math-cli -- sub 20 5
# 输出: 20 - 5 = 15
```

---

## 与其他语言对比

| 概念 | Rust Workspace | Node.js | PHP |
|------|----------------|---------|-----|
| 根配置 | `Cargo.toml` | `package.json` (workspaces) | `composer.json` |
| 子包 | crate (`Cargo.toml`) | package | package |
| 共享依赖 | `workspace.dependencies` | hoisting | path repository |
| 内部引用 | `path = "../other"` | `"other": "*"` | `"other": "@dev"` |

---

## 总结

- **Workspace** = 多个 crate 共享配置和编译输出
- **workspace.dependencies** = 统一依赖版本
- **workspace.package** = 统一 package 元数据
- **path 依赖** = crate 之间的引用
- **cargo -p** = 指定操作某个 crate

Workspace 是 Rust 项目规模化的关键，掌握它能让你的项目结构清晰、编译高效！

---

*下节课：条件编译与 Feature Flags*
