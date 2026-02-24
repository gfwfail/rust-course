# 第 100 课：std::panic — Panic 机制与恢复

🎉 **里程碑达成！第一百课！**

## 🎯 今日主题

Rust 有两种错误处理方式：**可恢复错误**（Result）和**不可恢复错误**（panic）。今天深入讲 panic 机制：什么时候用、怎么用、能不能恢复。

---

## 📚 Panic 是什么？

**Panic** 是 Rust 处理不可恢复错误的方式。当 panic 发生时：

1. 打印错误信息
2. **展开（unwind）**栈，逐层清理资源
3. 进程退出

```rust
fn main() {
    panic!("出大事了！");  // 程序崩溃，退出码非零
}
```

---

## 🔍 什么时候会 Panic？

### 1. 显式调用

```rust
panic!("自定义错误信息");
panic!("值是：{}", some_value);

// 常用的 panic 变体
todo!();         // 未实现的代码占位
unimplemented!(); // 明确表示不会实现
unreachable!();  // 逻辑上不可达的代码
```

### 2. 数组越界

```rust
let v = vec![1, 2, 3];
let x = v[100];  // panic: index out of bounds
```

### 3. unwrap / expect 失败

```rust
let x: Option<i32> = None;
x.unwrap();  // panic: called `unwrap()` on a `None` value

let y: Result<i32, &str> = Err("失败了");
y.expect("这里不应该失败");  // panic with custom message
```

### 4. 算术溢出（debug 模式）

```rust
let x: u8 = 255;
let y = x + 1;  // debug 模式 panic，release 模式 wrap
```

---

## 🛡️ 捕获 Panic：catch_unwind

有时候你**不想让 panic 终止整个程序**，可以用 `catch_unwind`：

```rust
use std::panic;

fn main() {
    let result = panic::catch_unwind(|| {
        println!("开始执行");
        panic!("出错了！");
        println!("这行不会执行");
    });

    match result {
        Ok(_) => println!("正常完成"),
        Err(e) => {
            // e 是 Box<dyn Any + Send>
            if let Some(s) = e.downcast_ref::<&str>() {
                println!("捕获到 panic: {}", s);
            } else if let Some(s) = e.downcast_ref::<String>() {
                println!("捕获到 panic: {}", s);
            } else {
                println!("捕获到未知 panic");
            }
        }
    }

    println!("程序继续运行！");
}
```

**输出**：
```
开始执行
捕获到 panic: 出错了！
程序继续运行！
```

---

## ⚠️ catch_unwind 的限制

### 1. 只能捕获 unwind，不能捕获 abort

```toml
# Cargo.toml
[profile.release]
panic = "abort"  # 这种模式下 panic 直接终止，无法捕获
```

### 2. 闭包必须是 UnwindSafe

```rust
use std::panic::{catch_unwind, AssertUnwindSafe};

let mut data = vec![1, 2, 3];

// ❌ 编译错误：&mut data 不是 UnwindSafe
// let result = catch_unwind(|| {
//     data.push(4);
//     panic!("oops");
// });

// ✅ 用 AssertUnwindSafe 包装
let result = catch_unwind(AssertUnwindSafe(|| {
    data.push(4);
    panic!("oops");
}));
```

### 3. 不适合常规错误处理

`catch_unwind` 主要用于：
- FFI 边界（防止 panic 跨越 C 边界，这是 UB）
- 线程池隔离（一个任务 panic 不影响其他任务）
- 测试框架

**不要用它替代 Result！**

---

## 🔧 自定义 Panic Hook

```rust
use std::panic;

fn main() {
    // 设置自定义 panic 处理器
    panic::set_hook(Box::new(|info| {
        // info: &PanicInfo
        
        // 获取 panic 位置
        if let Some(location) = info.location() {
            eprintln!(
                "💥 Panic 发生在 {}:{}:{}",
                location.file(),
                location.line(),
                location.column()
            );
        }

        // 获取 panic 消息
        if let Some(s) = info.payload().downcast_ref::<&str>() {
            eprintln!("消息: {}", s);
        }

        // 这里可以：发送告警、写日志、收集堆栈等
    }));

    panic!("测试 panic");
}
```

### 获取默认 hook

```rust
// 获取当前 hook（并恢复默认）
let default_hook = panic::take_hook();

// 链式调用：先自己处理，再调用默认
panic::set_hook(Box::new(move |info| {
    eprintln!("🚨 自定义处理");
    default_hook(info);  // 调用默认处理
}));
```

---

## 📊 Panic vs Abort 模式

| 特性 | Panic (unwind) | Abort |
|------|----------------|-------|
| 析构函数 | ✅ 会调用 Drop | ❌ 不调用 |
| catch_unwind | ✅ 可捕获 | ❌ 不可捕获 |
| 二进制大小 | 较大（展开表） | 较小 |
| 速度 | 稍慢 | 稍快 |

```toml
# Cargo.toml - 设置 abort 模式
[profile.release]
panic = "abort"
```

---

## 💡 Panic 最佳实践

### ✅ 应该 Panic 的情况

```rust
// 1. 逻辑错误 / 程序 bug
fn get_item(index: usize, items: &[i32]) -> i32 {
    items[index]  // 越界是调用者的 bug
}

// 2. 违反不变量
fn create_positive(n: i32) -> u32 {
    assert!(n > 0, "必须是正数");
    n as u32
}

// 3. 无法恢复的状态
fn init() {
    CONFIG.set(load_config()).expect("配置只能初始化一次");
}
```

### ❌ 不应该 Panic 的情况

```rust
// 1. 用户输入错误 → 用 Result
fn parse_port(s: &str) -> Result<u16, ParseIntError> {
    s.parse()  // 不要 unwrap！
}

// 2. 文件/网络操作 → 用 Result
fn read_config(path: &str) -> Result<Config, io::Error> {
    let content = fs::read_to_string(path)?;
    // ...
}

// 3. 可预期的失败
fn find_user(id: u64) -> Option<User> {
    // 找不到返回 None，不要 panic
}
```

---

## 🎯 实战：线程 Panic 隔离

```rust
use std::thread;

fn main() {
    let handles: Vec<_> = (0..3)
        .map(|i| {
            thread::spawn(move || {
                if i == 1 {
                    panic!("线程 {} 崩溃了！", i);
                }
                println!("线程 {} 完成", i);
                i * 10
            })
        })
        .collect();

    for (i, handle) in handles.into_iter().enumerate() {
        match handle.join() {
            Ok(result) => println!("线程 {} 返回: {}", i, result),
            Err(e) => {
                // e 是 Box<dyn Any + Send>
                if let Some(s) = e.downcast_ref::<&str>() {
                    println!("线程 {} panic: {}", i, s);
                }
            }
        }
    }

    println!("主线程继续运行");
}
```

**输出**：
```
线程 0 完成
线程 2 完成
线程 0 返回: 0
线程 1 panic: 线程 1 崩溃了！
线程 2 返回: 20
主线程继续运行
```

线程的 panic 不会影响主线程！

---

## 📝 小结

| 概念 | 说明 |
|------|------|
| `panic!()` | 触发不可恢复错误 |
| `todo!()`/`unimplemented!()` | 占位宏 |
| `unreachable!()` | 标记不可达代码 |
| `catch_unwind` | 捕获 panic（有限场景） |
| `set_hook` | 自定义 panic 处理 |
| `take_hook` | 获取/替换 hook |
| `AssertUnwindSafe` | 跳过 UnwindSafe 检查 |

**黄金法则**：
- 逻辑错误、bug → panic
- 预期内的失败 → Result/Option
- FFI 边界 → 必须 catch_unwind，不能让 panic 跨越

---

*下节预告：std::ffi — FFI 类型与 C 语言互操作*
