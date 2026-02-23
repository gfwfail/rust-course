# 第 93 课：std::env — 环境变量与命令行参数

## 什么是 std::env？

`std::env` 提供了与进程环境交互的功能：
- 读取/设置环境变量
- 获取命令行参数
- 获取当前工作目录
- 获取可执行文件路径

类比 PHP：相当于 `$_ENV`、`$_SERVER`、`$argv` 和 `getcwd()` 的组合。

## 环境变量操作

### 读取环境变量

```rust
use std::env;

fn main() {
    // 方式 1：返回 Result
    match env::var("HOME") {
        Ok(val) => println!("HOME = {}", val),
        Err(e) => println!("HOME 未设置: {}", e),
    }
    
    // 方式 2：带默认值（常用！）
    let port = env::var("PORT").unwrap_or("3000".to_string());
    println!("端口: {}", port);
    
    // 方式 3：检查是否存在
    if env::var("DEBUG").is_ok() {
        println!("调试模式已开启");
    }
}
```

### 遍历所有环境变量

```rust
use std::env;

fn main() {
    // vars() 返回迭代器
    for (key, value) in env::vars() {
        println!("{} = {}", key, value);
    }
    
    // 只看变量名
    let keys: Vec<_> = env::vars().map(|(k, _)| k).collect();
    println!("共有 {} 个环境变量", keys.len());
}
```

### 设置环境变量

```rust
use std::env;

fn main() {
    // 设置（只影响当前进程和子进程）
    env::set_var("MY_APP_DEBUG", "true");
    
    // 删除
    env::remove_var("MY_APP_DEBUG");
    
    // ⚠️ 注意：这些修改不会影响 shell 环境
}
```

## 命令行参数

### 基本用法

```rust
use std::env;

fn main() {
    // args() 返回迭代器，第一个是程序名
    let args: Vec<String> = env::args().collect();
    
    println!("程序名: {}", args[0]);
    println!("参数数量: {}", args.len() - 1);
    
    // 遍历参数（跳过程序名）
    for arg in args.iter().skip(1) {
        println!("参数: {}", arg);
    }
}
```

### 实用模式：解析简单参数

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    
    // 类似：./myapp input.txt output.txt
    if args.len() != 3 {
        eprintln!("用法: {} <输入文件> <输出文件>", args[0]);
        std::process::exit(1);
    }
    
    let input = &args[1];
    let output = &args[2];
    
    println!("输入: {}, 输出: {}", input, output);
}
```

### 处理非 UTF-8 参数

```rust
use std::env;
use std::ffi::OsString;

fn main() {
    // args_os() 返回 OsString，可处理任意编码
    let args: Vec<OsString> = env::args_os().collect();
    
    for arg in args {
        // 尝试转换为 String
        match arg.into_string() {
            Ok(s) => println!("有效 UTF-8: {}", s),
            Err(os) => println!("非 UTF-8: {:?}", os),
        }
    }
}
```

## 路径相关函数

```rust
use std::env;

fn main() -> std::io::Result<()> {
    // 当前工作目录
    let cwd = env::current_dir()?;
    println!("当前目录: {}", cwd.display());
    
    // 修改工作目录
    env::set_current_dir("/tmp")?;
    
    // 当前可执行文件路径
    let exe = env::current_exe()?;
    println!("可执行文件: {}", exe.display());
    
    // 临时目录
    let tmp = env::temp_dir();
    println!("临时目录: {}", tmp.display());
    
    Ok(())
}
```

## 实战示例：简易配置加载器

```rust
use std::env;

struct Config {
    host: String,
    port: u16,
    debug: bool,
}

impl Config {
    fn from_env() -> Self {
        Config {
            host: env::var("APP_HOST")
                .unwrap_or_else(|_| "127.0.0.1".to_string()),
            port: env::var("APP_PORT")
                .ok()
                .and_then(|p| p.parse().ok())
                .unwrap_or(8080),
            debug: env::var("APP_DEBUG")
                .map(|v| v == "1" || v.to_lowercase() == "true")
                .unwrap_or(false),
        }
    }
}

fn main() {
    // 测试：设置环境变量
    env::set_var("APP_PORT", "3000");
    env::set_var("APP_DEBUG", "true");
    
    let config = Config::from_env();
    
    println!("服务器: {}:{}", config.host, config.port);
    println!("调试模式: {}", config.debug);
}
```

## 注意事项

1. **线程安全**：`set_var` 和 `remove_var` 在多线程环境下不安全（会 panic 或数据竞争），尽量在程序启动时设置

2. **编码问题**：Windows 环境变量可能不是 UTF-8，用 `var_os()` / `args_os()` 更安全

3. **进程隔离**：修改只影响当前进程，不会改变 shell 环境

## 常用函数速查

| 函数 | 说明 |
|------|------|
| `var(key)` | 读取环境变量，返回 `Result<String>` |
| `var_os(key)` | 读取环境变量，返回 `Option<OsString>` |
| `set_var(key, val)` | 设置环境变量 |
| `remove_var(key)` | 删除环境变量 |
| `vars()` | 遍历所有环境变量 |
| `args()` | 获取命令行参数（UTF-8） |
| `args_os()` | 获取命令行参数（OsString） |
| `current_dir()` | 获取当前工作目录 |
| `set_current_dir(path)` | 修改当前工作目录 |
| `current_exe()` | 获取可执行文件路径 |
| `temp_dir()` | 获取临时目录 |

## 课后练习

写一个程序，实现：
1. 接收命令行参数指定文件名
2. 如果设置了 `VERBOSE=1`，打印详细信息
3. 读取 `DEFAULT_PATH` 环境变量作为默认路径

---

*下一课：std::process — 进程控制与子进程*
