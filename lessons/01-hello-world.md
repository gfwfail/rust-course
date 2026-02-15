# 第1课：Hello World 与基础概念

## Rust 是什么？

Rust 是一门系统级编程语言，主打：
- **内存安全**（没有垃圾回收，但编译器帮你管内存）
- **零成本抽象**（高级特性不牺牲性能）
- **并发安全**（编译期防止数据竞争）

## 安装 Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

安装完后验证：
```bash
rustc --version
cargo --version
```

## Hello World

创建项目：
```bash
cargo new hello_rust
cd hello_rust
```

`src/main.rs` 内容：
```rust
fn main() {
    println!("Hello, World!");
}
```

运行：
```bash
cargo run
```

## 代码解析

- `fn main()` — 程序入口函数
- `println!` — 这是一个**宏**（注意感叹号 `!`），不是普通函数
- 语句结尾要加分号 `;`

## 变量初体验

```rust
fn main() {
    let name = "Rust学习小组";
    println!("Hello, {}!", name);
}
```

`{}` 是占位符，会被 `name` 的值替换。

## Cargo 常用命令

| 命令 | 作用 |
|------|------|
| `cargo new` | 创建新项目 |
| `cargo build` | 编译项目 |
| `cargo run` | 编译并运行 |
| `cargo check` | 快速检查代码（不生成可执行文件） |
| `cargo test` | 运行测试 |

---

**下节课**：变量与数据类型
