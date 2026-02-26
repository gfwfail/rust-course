# 第 102 课：std::panic — Panic 处理机制

## 什么是 Panic？

Rust 有两种错误处理方式：

1. **可恢复错误**：用 `Result<T, E>`，调用者决定如何处理
2. **不可恢复错误**：用 `panic!`，程序崩溃

**类比其他语言：**
```
PHP:   throw new Exception() + try/catch
JS:    throw Error() + try/catch  
Rust:  panic!() — 默认不可捕获！
```

---

## 基本用法

```rust
// 直接 panic
panic!("出大事了！");

// 带格式化信息
panic!("用户 {} 不存在", user_id);

// 常见触发 panic 的情况
let v = vec![1, 2, 3];
let _ = v[10];  // 越界访问 → panic!

let x: Option<i32> = None;
x.unwrap();  // None 上调用 unwrap → panic!
```

---

## Panic 的两种模式

在 `Cargo.toml` 中配置：

```toml
[profile.release]
panic = "unwind"   # 默认：栈展开，可以捕获
# panic = "abort"  # 直接终止进程，不可捕获
```

**unwind（默认）：**
- 逐层回退栈帧，执行每一层的析构函数（Drop）
- 可以被 `catch_unwind` 捕获
- 二进制文件稍大

**abort：**
- 直接调用 `abort()`，进程立即终止
- 不执行任何清理
- 二进制更小，嵌入式常用

---

## catch_unwind — 捕获 Panic

```rust
use std::panic;

fn main() {
    // 捕获 panic，返回 Result
    let result = panic::catch_unwind(|| {
        println!("正常执行...");
        panic!("💥 爆炸！");
    });
    
    match result {
        Ok(_) => println!("顺利完成"),
        Err(e) => {
            // 尝试获取 panic 信息
            if let Some(s) = e.downcast_ref::<&str>() {
                println!("捕获到 panic: {}", s);
            } else if let Some(s) = e.downcast_ref::<String>() {
                println!("捕获到 panic: {}", s);
            }
        }
    }
    
    println!("程序继续运行！");
}
```

输出：
```
正常执行...
捕获到 panic: 💥 爆炸！
程序继续运行！
```

---

## 使用场景

**什么时候用 `catch_unwind`？**

```rust
// ✅ 场景1：线程池 worker，不能让一个任务崩掉整个池
fn run_task(task: impl FnOnce() + UnwindSafe) {
    let _ = panic::catch_unwind(task);
    // 任务 panic 了也能继续处理下一个
}

// ✅ 场景2：FFI 边界，防止 panic 跨越 FFI
#[no_mangle]
pub extern "C" fn rust_function() -> i32 {
    match panic::catch_unwind(|| do_work()) {
        Ok(v) => v,
        Err(_) => -1,  // 返回错误码，而不是让 panic 穿透到 C
    }
}

// ❌ 不要用来做常规错误处理！用 Result
fn bad_idea() {
    let result = panic::catch_unwind(|| {
        let x: i32 = "abc".parse().unwrap();  // ← 这样做是错的！
        x
    });
}

// ✅ 正确做法
fn good_idea() -> Result<i32, ParseIntError> {
    "abc".parse()  // 返回 Result，让调用者处理
}
```

---

## panic::set_hook — 自定义 Panic 处理

```rust
use std::panic;

fn main() {
    // 设置自定义 panic 钩子
    panic::set_hook(Box::new(|info| {
        // info: PanicInfo，包含 panic 的详细信息
        
        // 获取 panic 消息
        let msg = if let Some(s) = info.payload().downcast_ref::<&str>() {
            s.to_string()
        } else if let Some(s) = info.payload().downcast_ref::<String>() {
            s.clone()
        } else {
            "Unknown panic".to_string()
        };
        
        // 获取 panic 位置
        let location = info.location()
            .map(|loc| format!("{}:{}:{}", loc.file(), loc.line(), loc.column()))
            .unwrap_or_else(|| "unknown".to_string());
        
        // 自定义输出（比如发送到日志服务）
        eprintln!("🔥 PANIC at {}", location);
        eprintln!("   Message: {}", msg);
    }));
    
    panic!("测试 panic");
}
```

---

## resume_unwind — 重新触发 Panic

```rust
use std::panic;

fn main() {
    let result = panic::catch_unwind(|| {
        panic!("原始 panic");
    });
    
    if let Err(e) = result {
        println!("捕获到了，做一些清理...");
        
        // 清理完毕，重新 panic
        panic::resume_unwind(e);
    }
}
```

---

## UnwindSafe trait

`catch_unwind` 要求闭包是 `UnwindSafe` 的：

```rust
use std::panic::{catch_unwind, UnwindSafe, AssertUnwindSafe};

// ❌ 这样编译不过
fn bad() {
    let mut x = 0;
    let _ = catch_unwind(|| {
        x += 1;  // 捕获可变引用，不是 UnwindSafe
    });
}

// ✅ 用 AssertUnwindSafe 包装（你保证安全）
fn good() {
    let mut x = 0;
    let _ = catch_unwind(AssertUnwindSafe(|| {
        x += 1;
    }));
    println!("x = {}", x);  // 可能是 0 或 1，取决于是否 panic
}
```

**为什么需要 UnwindSafe？**

Panic 可能在任意时刻发生，如果捕获后继续使用被修改到一半的数据，可能导致不一致状态。`UnwindSafe` 是编译器帮你检查的安全护栏。

---

## 实战：优雅的 panic 日志

```rust
use std::panic;
use std::backtrace::Backtrace;

fn setup_panic_handler() {
    panic::set_hook(Box::new(|info| {
        let backtrace = Backtrace::capture();
        
        let thread = std::thread::current();
        let thread_name = thread.name().unwrap_or("unnamed");
        
        let msg = match info.payload().downcast_ref::<&str>() {
            Some(s) => *s,
            None => match info.payload().downcast_ref::<String>() {
                Some(s) => s.as_str(),
                None => "Unknown panic message",
            },
        };
        
        eprintln!("╔════════════════════════════════════════╗");
        eprintln!("║           🔥 PANIC OCCURRED 🔥          ║");
        eprintln!("╠════════════════════════════════════════╣");
        eprintln!("║ Thread: {:<30} ║", thread_name);
        if let Some(loc) = info.location() {
            eprintln!("║ Location: {}:{}:{}", loc.file(), loc.line(), loc.column());
        }
        eprintln!("║ Message: {}", msg);
        eprintln!("╠════════════════════════════════════════╣");
        eprintln!("{}", backtrace);
        eprintln!("╚════════════════════════════════════════╝");
    }));
}

fn main() {
    setup_panic_handler();
    
    let _ = std::thread::Builder::new()
        .name("worker-1".into())
        .spawn(|| {
            panic!("Something went wrong!");
        })
        .unwrap()
        .join();
}
```

---

## 要点总结

| 函数/宏 | 作用 |
|--------|------|
| `panic!()` | 触发 panic |
| `catch_unwind()` | 捕获 panic（unwind 模式） |
| `resume_unwind()` | 重新触发捕获的 panic |
| `set_hook()` | 设置全局 panic 处理钩子 |
| `take_hook()` | 取回之前设置的钩子 |
| `AssertUnwindSafe` | 声明类型是 unwind safe |

**记住：**
- `panic!` 用于不可恢复的错误
- `catch_unwind` 主要用于 FFI 边界和线程隔离
- **常规错误处理用 Result，不要用 catch_unwind！**

---

*下节课：std::hint — 编译器优化提示*
