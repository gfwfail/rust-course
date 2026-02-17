# 第 41 课：CLI 工具开发 (clap)

> 日期：2026-02-17  
> 主题：用 clap 构建命令行工具

---

## 为什么用 Rust 写 CLI？

- **单文件分发**：编译成一个二进制，不需要运行时
- **跨平台**：一份代码，编译出 Windows/Mac/Linux 版本
- **启动快**：毫秒级启动，不像 Node/Python 要加载解释器
- **类型安全**：参数解析编译期检查

---

## clap 简介

`clap` 是 Rust 生态最流行的命令行参数解析库，名字来自 "Command Line Argument Parser"。

```bash
cargo add clap --features derive
```

---

## 基础示例：问候程序

```rust
use clap::Parser;

/// 一个简单的问候程序
#[derive(Parser)]
#[command(name = "greet")]
#[command(version = "1.0")]
#[command(about = "向某人问好")]
struct Cli {
    /// 要问候的名字
    name: String,
    
    /// 问候的次数
    #[arg(short, long, default_value_t = 1)]
    count: u8,
}

fn main() {
    let cli = Cli::parse();
    
    for _ in 0..cli.count {
        println!("你好, {}!", cli.name);
    }
}
```

运行效果：

```bash
$ cargo run -- Alice
你好, Alice!

$ cargo run -- Alice --count 3
你好, Alice!
你好, Alice!
你好, Alice!

$ cargo run -- --help
一个简单的问候程序

Usage: greet [OPTIONS] <NAME>

Arguments:
  <NAME>  要问候的名字

Options:
  -c, --count <COUNT>  问候的次数 [default: 1]
  -h, --help           Print help
  -V, --version        Print version
```

---

## 参数类型

### 1. 位置参数 (Positional Arguments)

```rust
#[derive(Parser)]
struct Cli {
    /// 输入文件
    input: String,
    
    /// 输出文件（可选）
    output: Option<String>,
}
```

### 2. 选项参数 (Options)

```rust
#[derive(Parser)]
struct Cli {
    /// 配置文件路径
    #[arg(short, long)]
    config: Option<String>,
    
    /// 端口号
    #[arg(short, long, default_value_t = 8080)]
    port: u16,
}
```

- `short` = `-c`
- `long` = `--config`
- 可以自定义：`#[arg(short = 'p', long = "port")]`

### 3. 标志参数 (Flags)

```rust
#[derive(Parser)]
struct Cli {
    /// 启用详细输出
    #[arg(short, long)]
    verbose: bool,
    
    /// 增加详细程度（-v, -vv, -vvv）
    #[arg(short = 'v', long, action = clap::ArgAction::Count)]
    verbosity: u8,
}
```

---

## 子命令 (Subcommands)

像 `git clone`、`cargo build` 这样的多级命令：

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "myapp")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// 添加新项目
    Add {
        /// 项目名称
        name: String,
    },
    /// 列出所有项目
    List {
        /// 显示详细信息
        #[arg(short, long)]
        verbose: bool,
    },
    /// 删除项目
    Remove {
        /// 项目 ID
        id: u32,
        /// 强制删除
        #[arg(short, long)]
        force: bool,
    },
}

fn main() {
    let cli = Cli::parse();
    
    match cli.command {
        Commands::Add { name } => {
            println!("添加项目: {}", name);
        }
        Commands::List { verbose } => {
            if verbose {
                println!("详细列表...");
            } else {
                println!("简单列表...");
            }
        }
        Commands::Remove { id, force } => {
            if force {
                println!("强制删除项目 {}", id);
            } else {
                println!("删除项目 {} (需确认)", id);
            }
        }
    }
}
```

使用：

```bash
$ myapp add todo-app
$ myapp list --verbose
$ myapp remove 123 --force
```

---

## 参数验证

```rust
use std::path::PathBuf;

#[derive(Parser)]
struct Cli {
    /// 输入文件（必须存在）
    #[arg(value_parser = validate_file_exists)]
    input: PathBuf,
    
    /// 端口号（1024-65535）
    #[arg(short, long, value_parser = clap::value_parser!(u16).range(1024..=65535))]
    port: u16,
    
    /// 日志级别
    #[arg(short, long, value_parser = ["debug", "info", "warn", "error"])]
    level: String,
}

fn validate_file_exists(s: &str) -> Result<PathBuf, String> {
    let path = PathBuf::from(s);
    if path.exists() {
        Ok(path)
    } else {
        Err(format!("文件不存在: {}", s))
    }
}
```

---

## 环境变量支持

```rust
#[derive(Parser)]
struct Cli {
    /// 数据库 URL
    #[arg(long, env = "DATABASE_URL")]
    database_url: String,
    
    /// API 密钥
    #[arg(long, env = "API_KEY")]
    api_key: Option<String>,
}
```

优先级：**命令行参数 > 环境变量 > 默认值**

---

## 完整实战：文件搜索工具

```rust
use clap::{Parser, ValueEnum};
use std::path::PathBuf;

#[derive(Parser)]
#[command(name = "search")]
#[command(about = "在文件中搜索内容")]
struct Cli {
    /// 搜索模式
    pattern: String,
    
    /// 搜索路径
    #[arg(default_value = ".")]
    path: PathBuf,
    
    /// 忽略大小写
    #[arg(short, long)]
    ignore_case: bool,
    
    /// 输出格式
    #[arg(short, long, value_enum, default_value_t = OutputFormat::Text)]
    format: OutputFormat,
    
    /// 最大搜索深度
    #[arg(short, long)]
    depth: Option<usize>,
    
    /// 文件扩展名过滤
    #[arg(short, long)]
    extension: Vec<String>,
}

#[derive(Clone, ValueEnum)]
enum OutputFormat {
    Text,
    Json,
    Csv,
}

fn main() {
    let cli = Cli::parse();
    
    println!("搜索模式: {}", cli.pattern);
    println!("搜索路径: {:?}", cli.path);
    println!("忽略大小写: {}", cli.ignore_case);
    
    if let Some(depth) = cli.depth {
        println!("最大深度: {}", depth);
    }
    
    if !cli.extension.is_empty() {
        println!("扩展名过滤: {:?}", cli.extension);
    }
}
```

使用示例：

```bash
# 搜索当前目录
$ search "TODO"

# 搜索指定目录，忽略大小写
$ search "error" ./src -i

# 只搜索 .rs 文件，输出 JSON
$ search "fn main" -e rs -e toml --format json

# 限制深度
$ search "password" /etc --depth 2
```

---

## 对比 Laravel Artisan

| 概念 | Laravel Artisan | clap |
|------|-----------------|------|
| 定义命令 | `$signature` | `#[derive(Parser)]` |
| 位置参数 | `{name}` | 字段无 `#[arg]` |
| 可选参数 | `{name?}` | `Option<T>` |
| 选项 | `{--option}` | `#[arg(long)]` |
| 带值选项 | `{--option=}` | `#[arg(long)]` |
| 默认值 | `{--option=default}` | `default_value` |
| 描述 | `$description` | doc comment `///` |

---

## 小结

1. **clap 用 derive 宏**：定义结构体，自动生成解析逻辑
2. **参数类型**：位置参数、选项 (`--opt`)、标志 (`--flag`)
3. **子命令**：用 enum + `#[derive(Subcommand)]`
4. **验证**：用 `value_parser` 做类型检查和范围限制
5. **环境变量**：用 `env` 属性支持配置优先级

---

## 作业

用 clap 写一个简单的 TODO CLI 工具，支持 `add`、`list`、`done`、`remove` 子命令。

---

## 参考资源

- [clap 官方文档](https://docs.rs/clap/latest/clap/)
- [clap derive 教程](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html)
- [clap cookbook](https://docs.rs/clap/latest/clap/_derive/_cookbook/index.html)
