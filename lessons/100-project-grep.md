# 第 100 课：实战项目 — 用纯标准库实现 minigrep 🎉

恭喜你走到第 100 课！今天我们来做一个综合实战项目：用纯标准库实现一个简化版的 `grep` 命令行工具。这个项目会用到我们学过的大部分知识。

---

## 🎯 项目目标

```bash
# 用法
minigrep <pattern> <file_path>

# 示例
minigrep "fn main" src/main.rs
# 输出所有包含 "fn main" 的行
```

---

## 📁 项目结构

```
minigrep/
├── src/
│   ├── main.rs    # 入口：解析参数、调用库
│   └── lib.rs     # 核心逻辑
```

---

## 🔧 第一步：解析命令行参数 (std::env)

```rust
// src/main.rs
use std::env;
use std::process;

fn main() {
    // 收集命令行参数
    let args: Vec<String> = env::args().collect();
    
    // 解析参数（跳过第一个，那是程序名）
    let config = Config::parse(&args).unwrap_or_else(|err| {
        // 错误输出到 stderr
        eprintln!("参数错误: {}", err);
        process::exit(1);
    });
    
    println!("搜索: '{}' 在文件: {}", config.pattern, config.file_path);
    
    // 运行主逻辑
    if let Err(e) = minigrep::run(config) {
        eprintln!("运行错误: {}", e);
        process::exit(1);
    }
}

pub struct Config {
    pub pattern: String,
    pub file_path: String,
    pub ignore_case: bool,
}

impl Config {
    pub fn parse(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("用法: minigrep <pattern> <file_path>");
        }
        
        let pattern = args[1].clone();
        let file_path = args[2].clone();
        
        // 通过环境变量控制大小写敏感
        let ignore_case = env::var("IGNORE_CASE").is_ok();
        
        Ok(Config {
            pattern,
            file_path,
            ignore_case,
        })
    }
}
```

**知识点回顾**：
- `env::args()` — 获取命令行参数 (第 93 课)
- `env::var()` — 读取环境变量
- `process::exit()` — 退出程序 (第 94 课)
- `eprintln!` — 输出到 stderr

---

## 📖 第二步：读取文件 (std::fs, std::io)

```rust
// src/lib.rs
use std::fs;
use std::error::Error;

pub use crate::Config;

// 使用 Box<dyn Error> 作为统一错误类型
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    // 读取整个文件内容
    let contents = fs::read_to_string(&config.file_path)?;
    
    // 搜索并打印结果
    let results = if config.ignore_case {
        search_case_insensitive(&config.pattern, &contents)
    } else {
        search(&config.pattern, &contents)
    };
    
    for (line_num, line) in results {
        println!("{}: {}", line_num, line);
    }
    
    Ok(())
}
```

**知识点回顾**：
- `fs::read_to_string()` — 读取文件 (第 91 课)
- `Box<dyn Error>` — trait object 处理多种错误类型
- `?` 操作符 — 错误传播

---

## 🔍 第三步：搜索逻辑 (迭代器)

```rust
// src/lib.rs (继续)

/// 大小写敏感搜索
pub fn search<'a>(pattern: &str, contents: &'a str) -> Vec<(usize, &'a str)> {
    contents
        .lines()                          // 按行迭代
        .enumerate()                       // 加上行号 (0, line)
        .filter(|(_, line)| line.contains(pattern))  // 过滤匹配行
        .map(|(i, line)| (i + 1, line))   // 行号从 1 开始
        .collect()                         // 收集结果
}

/// 大小写不敏感搜索
pub fn search_case_insensitive<'a>(
    pattern: &str, 
    contents: &'a str
) -> Vec<(usize, &'a str)> {
    let pattern = pattern.to_lowercase();
    
    contents
        .lines()
        .enumerate()
        .filter(|(_, line)| line.to_lowercase().contains(&pattern))
        .map(|(i, line)| (i + 1, line))
        .collect()
}
```

**知识点回顾**：
- 迭代器链式调用 (第 84 课)
- `enumerate()` — 带索引迭代
- `filter()` + `map()` — 函数式数据处理
- 生命周期标注 `'a` — 确保返回的 `&str` 引用有效

---

## ✅ 第四步：单元测试 (std::test)

```rust
// src/lib.rs (继续)

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn case_sensitive() {
        let pattern = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Duct tape.";
        
        let results = search(pattern, contents);
        assert_eq!(results, vec![(2, "safe, fast, productive.")]);
    }
    
    #[test]
    fn case_insensitive() {
        let pattern = "rUsT";
        let contents = "\
Rust:
safe, fast, productive.
Trust me.";
        
        let results = search_case_insensitive(pattern, contents);
        assert_eq!(results, vec![
            (1, "Rust:"),
            (3, "Trust me."),
        ]);
    }
    
    #[test]
    fn no_match() {
        let pattern = "xyz";
        let contents = "hello\nworld";
        
        let results = search(pattern, contents);
        assert!(results.is_empty());
    }
}
```

运行测试：
```bash
cargo test
```

---

## 🎨 第五步：彩色输出 (bonus，纯 ANSI)

不用第三方库也能做彩色输出！用 ANSI 转义码：

```rust
// 彩色常量
const RED: &str = "\x1b[31m";
const GREEN: &str = "\x1b[32m";
const YELLOW: &str = "\x1b[33m";
const RESET: &str = "\x1b[0m";
const BOLD: &str = "\x1b[1m";

fn print_match(line_num: usize, line: &str, pattern: &str) {
    // 高亮匹配的部分
    let highlighted = line.replace(
        pattern, 
        &format!("{}{}{}{}", RED, BOLD, pattern, RESET)
    );
    
    println!("{}{:4}{}: {}", GREEN, line_num, RESET, highlighted);
}
```

效果：
```
   1: fn main() {        # "main" 会显示红色高亮
   5: fn main() -> i32 { # "main" 会显示红色高亮
```

---

## 📦 完整代码

### src/main.rs
```rust
use std::env;
use std::process;
use minigrep::Config;

fn main() {
    let args: Vec<String> = env::args().collect();
    
    let config = Config::parse(&args).unwrap_or_else(|err| {
        eprintln!("错误: {}", err);
        process::exit(1);
    });
    
    if let Err(e) = minigrep::run(config) {
        eprintln!("错误: {}", e);
        process::exit(1);
    }
}
```

### src/lib.rs
```rust
use std::env;
use std::fs;
use std::error::Error;

pub struct Config {
    pub pattern: String,
    pub file_path: String,
    pub ignore_case: bool,
}

impl Config {
    pub fn parse(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("用法: minigrep <pattern> <file>");
        }
        
        Ok(Config {
            pattern: args[1].clone(),
            file_path: args[2].clone(),
            ignore_case: env::var("IGNORE_CASE").is_ok(),
        })
    }
}

pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(&config.file_path)?;
    
    let results = if config.ignore_case {
        search_case_insensitive(&config.pattern, &contents)
    } else {
        search(&config.pattern, &contents)
    };
    
    if results.is_empty() {
        println!("没有找到匹配项");
    } else {
        for (num, line) in results {
            println!("{:4}: {}", num, line);
        }
    }
    
    Ok(())
}

pub fn search<'a>(pattern: &str, contents: &'a str) -> Vec<(usize, &'a str)> {
    contents
        .lines()
        .enumerate()
        .filter(|(_, line)| line.contains(pattern))
        .map(|(i, line)| (i + 1, line))
        .collect()
}

pub fn search_case_insensitive<'a>(
    pattern: &str,
    contents: &'a str,
) -> Vec<(usize, &'a str)> {
    let pattern = pattern.to_lowercase();
    contents
        .lines()
        .enumerate()
        .filter(|(_, line)| line.to_lowercase().contains(&pattern))
        .map(|(i, line)| (i + 1, line))
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn case_sensitive() {
        let results = search("duct", "productive\nDuct");
        assert_eq!(results, vec![(1, "productive")]);
    }
    
    #[test]
    fn case_insensitive() {
        let results = search_case_insensitive("rUsT", "Rust\nTrust");
        assert_eq!(results, vec![(1, "Rust"), (2, "Trust")]);
    }
}
```

---

## 🚀 运行效果

```bash
# 编译
cargo build --release

# 基本搜索
./target/release/minigrep "fn" src/lib.rs

# 忽略大小写
IGNORE_CASE=1 ./target/release/minigrep "rust" README.md

# 配合管道使用
cat file.txt | ./target/release/minigrep "pattern" /dev/stdin
```

---

## 🧠 知识点总结

这个项目用到了：

| 模块 | 用途 | 课程 |
|------|------|------|
| `std::env` | 命令行参数、环境变量 | 第 93 课 |
| `std::fs` | 文件读取 | 第 91 课 |
| `std::process` | 进程控制、退出码 | 第 94 课 |
| `std::error` | 错误处理 trait | 基础 |
| 迭代器 | lines, enumerate, filter, map | 第 84 课 |
| 生命周期 | 返回引用的正确标注 | 基础 |
| 单元测试 | #[test], assert_eq! | 基础 |

---

## 🎓 课后挑战

1. **支持正则表达式**：用标准库的模式匹配（不用 regex crate）
2. **递归搜索目录**：用 `fs::read_dir()` 遍历文件夹
3. **显示上下文**：像 `grep -C 2` 一样显示匹配行前后各 2 行
4. **统计模式**：只输出匹配数量（如 `grep -c`）
5. **多线程搜索**：用 `std::thread` 并行搜索多个文件

---

## 🎉 里程碑感言

100 课了！从最基础的变量、类型，到所有权、生命周期，再到标准库的各个模块，你已经建立了扎实的 Rust 基础。

接下来我们会继续深入语言特性和高级主题。Keep coding! 🦀

---

*第 100 课完 — 里程碑达成！*
