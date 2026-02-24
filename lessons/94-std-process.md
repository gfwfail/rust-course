# 第 94 课：std::process — 进程控制与子进程

## 什么是 std::process？

`std::process` 提供了与操作系统进程交互的功能：
- 启动和控制子进程
- 捕获子进程的输出
- 终止当前进程
- 获取退出状态

类比 PHP：相当于 `exec()`、`shell_exec()`、`proc_open()` 和 `exit()` 的组合，但功能更强大。

## 退出当前进程

### exit() — 立即退出

```rust
use std::process;

fn main() {
    // 检查某个条件
    if !check_requirements() {
        eprintln!("错误：缺少必要条件");
        process::exit(1);  // 非零表示错误
    }
    
    println!("程序正常运行...");
    // 隐式 exit(0)
}

fn check_requirements() -> bool {
    false
}
```

### abort() — 异常终止

```rust
use std::process;

fn main() {
    // 遇到不可恢复的错误时使用
    if critical_error() {
        process::abort();  // 立即终止，不运行析构函数
    }
}
```

### ExitCode — Rust 1.61+ 的新方式

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    if let Err(e) = run() {
        eprintln!("错误: {}", e);
        return ExitCode::FAILURE;
    }
    ExitCode::SUCCESS
}

fn run() -> Result<(), &'static str> {
    Err("模拟错误")
}
```

## 运行子进程：Command

`Command` 是运行外部程序的核心结构体。

### 基本用法

```rust
use std::process::Command;

fn main() {
    // 最简单的方式：运行命令，等待完成
    let status = Command::new("echo")
        .arg("Hello, Rust!")
        .status()
        .expect("执行失败");
    
    println!("退出状态: {}", status);
    println!("是否成功: {}", status.success());
}
```

### 捕获输出

```rust
use std::process::Command;

fn main() {
    // output() 捕获 stdout 和 stderr
    let output = Command::new("ls")
        .arg("-la")
        .output()
        .expect("执行失败");
    
    // 退出状态
    println!("成功: {}", output.status.success());
    
    // stdout（字节转字符串）
    let stdout = String::from_utf8_lossy(&output.stdout);
    println!("输出:\n{}", stdout);
    
    // stderr
    if !output.stderr.is_empty() {
        let stderr = String::from_utf8_lossy(&output.stderr);
        eprintln!("错误:\n{}", stderr);
    }
}
```

### 传递多个参数

```rust
use std::process::Command;

fn main() {
    // 方式 1：链式 .arg()
    Command::new("git")
        .arg("commit")
        .arg("-m")
        .arg("提交信息")
        .status()
        .unwrap();
    
    // 方式 2：.args() 传数组
    Command::new("git")
        .args(["commit", "-m", "提交信息"])
        .status()
        .unwrap();
}
```

### 设置工作目录和环境变量

```rust
use std::process::Command;

fn main() {
    let output = Command::new("pwd")
        .current_dir("/tmp")  // 设置工作目录
        .env("MY_VAR", "hello")  // 添加环境变量
        .env_remove("PATH")  // 移除环境变量
        .env_clear()  // 清除所有环境变量
        .envs([("KEY1", "val1"), ("KEY2", "val2")])  // 批量设置
        .output()
        .unwrap();
    
    println!("{}", String::from_utf8_lossy(&output.stdout));
}
```

## 控制输入输出：Stdio

### 重定向到 null

```rust
use std::process::{Command, Stdio};

fn main() {
    // 静默执行，丢弃所有输出
    Command::new("ls")
        .stdout(Stdio::null())
        .stderr(Stdio::null())
        .status()
        .unwrap();
}
```

### 管道：写入 stdin

```rust
use std::process::{Command, Stdio};
use std::io::Write;

fn main() {
    // 启动进程，准备写入 stdin
    let mut child = Command::new("cat")
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .spawn()
        .expect("启动失败");
    
    // 写入数据到 stdin
    if let Some(ref mut stdin) = child.stdin {
        stdin.write_all(b"Hello from Rust!\n").unwrap();
    }
    
    // 等待完成并获取输出
    let output = child.wait_with_output().unwrap();
    println!("{}", String::from_utf8_lossy(&output.stdout));
}
```

### 管道链：进程间通信

```rust
use std::process::{Command, Stdio};

fn main() {
    // 模拟：echo "hello\nworld" | grep "world"
    
    // 第一个进程
    let echo = Command::new("echo")
        .arg("hello\nworld")
        .stdout(Stdio::piped())
        .spawn()
        .unwrap();
    
    // 第二个进程，stdin 连接到第一个的 stdout
    let grep = Command::new("grep")
        .arg("world")
        .stdin(echo.stdout.unwrap())
        .output()
        .unwrap();
    
    println!("{}", String::from_utf8_lossy(&grep.stdout));
}
```

## 异步执行：spawn()

`spawn()` 启动进程但不等待完成。

```rust
use std::process::Command;
use std::thread;
use std::time::Duration;

fn main() {
    // 启动后台进程
    let mut child = Command::new("sleep")
        .arg("5")
        .spawn()
        .expect("启动失败");
    
    println!("子进程 PID: {}", child.id());
    
    // 做其他事情...
    println!("主程序继续执行...");
    thread::sleep(Duration::from_secs(1));
    
    // 检查是否完成（不阻塞）
    match child.try_wait() {
        Ok(Some(status)) => println!("已完成: {}", status),
        Ok(None) => println!("还在运行..."),
        Err(e) => println!("错误: {}", e),
    }
    
    // 等待完成（阻塞）
    let status = child.wait().unwrap();
    println!("最终状态: {}", status);
}
```

### 终止子进程

```rust
use std::process::Command;
use std::thread;
use std::time::Duration;

fn main() {
    let mut child = Command::new("sleep")
        .arg("100")
        .spawn()
        .unwrap();
    
    thread::sleep(Duration::from_secs(1));
    
    // 强制终止
    child.kill().expect("终止失败");
    println!("子进程已终止");
    
    // 清理（避免僵尸进程）
    child.wait().unwrap();
}
```

## 实战示例：命令执行器

```rust
use std::process::{Command, Stdio};
use std::io::{BufRead, BufReader};

/// 执行命令并实时打印输出
fn run_command(program: &str, args: &[&str]) -> std::io::Result<bool> {
    let mut child = Command::new(program)
        .args(args)
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()?;
    
    // 实时读取 stdout
    if let Some(stdout) = child.stdout.take() {
        let reader = BufReader::new(stdout);
        for line in reader.lines() {
            println!("[stdout] {}", line?);
        }
    }
    
    let status = child.wait()?;
    Ok(status.success())
}

fn main() {
    match run_command("ls", &["-la"]) {
        Ok(true) => println!("\n✓ 命令成功"),
        Ok(false) => println!("\n✗ 命令失败"),
        Err(e) => eprintln!("执行错误: {}", e),
    }
}
```

## 跨平台注意事项

### Windows vs Unix

```rust
use std::process::Command;

fn main() {
    // Unix: 直接运行
    #[cfg(unix)]
    let output = Command::new("ls").output();
    
    // Windows: 需要 cmd /C
    #[cfg(windows)]
    let output = Command::new("cmd")
        .args(["/C", "dir"])
        .output();
}
```

### 执行 shell 命令

```rust
use std::process::Command;

fn shell(cmd: &str) -> std::io::Result<String> {
    let output = if cfg!(target_os = "windows") {
        Command::new("cmd")
            .args(["/C", cmd])
            .output()?
    } else {
        Command::new("sh")
            .args(["-c", cmd])
            .output()?
    };
    
    Ok(String::from_utf8_lossy(&output.stdout).to_string())
}

fn main() {
    let result = shell("echo $HOME").unwrap();
    println!("{}", result);
}
```

## 常用 API 速查

| 方法 | 说明 |
|------|------|
| `Command::new(program)` | 创建新命令 |
| `.arg(arg)` | 添加单个参数 |
| `.args(args)` | 添加多个参数 |
| `.current_dir(dir)` | 设置工作目录 |
| `.env(key, val)` | 设置环境变量 |
| `.stdin(cfg)` | 配置 stdin |
| `.stdout(cfg)` | 配置 stdout |
| `.stderr(cfg)` | 配置 stderr |
| `.status()` | 运行并返回退出状态 |
| `.output()` | 运行并捕获输出 |
| `.spawn()` | 启动但不等待 |
| `child.wait()` | 等待子进程完成 |
| `child.try_wait()` | 非阻塞检查 |
| `child.kill()` | 终止子进程 |
| `child.id()` | 获取 PID |

## 课后练习

写一个简易的命令执行器：
1. 从命令行参数读取要执行的命令
2. 设置超时（5 秒），超时则 kill 进程
3. 打印 stdout、stderr 和退出状态

提示：使用 `spawn()` + `try_wait()` + `sleep` 循环实现超时。

---

*下一课：std::thread — 多线程基础*
