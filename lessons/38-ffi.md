# 第38课：FFI (Foreign Function Interface) - 与 C 语言交互

## 📚 什么是 FFI？

**FFI** 让 Rust 可以调用其他语言（主要是 C）的代码，也可以让其他语言调用 Rust。

这是 Rust 能够：
- 复用几十年积累的 C 库（OpenSSL、SQLite、zlib...）
- 嵌入到 Python/Ruby/Node 作为扩展
- 在操作系统底层工作

---

## 🎯 从 Rust 调用 C 函数

### 基本语法

```rust
// 声明外部 C 函数
extern "C" {
    fn abs(input: i32) -> i32;
    fn strlen(s: *const i8) -> usize;
}

fn main() {
    // 调用外部函数必须在 unsafe 块中
    unsafe {
        println!("abs(-5) = {}", abs(-5));
    }
}
```

### 为什么需要 unsafe？

编译器无法验证外部代码的：
- 内存安全性
- 函数签名是否正确
- 空指针处理
- 线程安全性

---

## 🔧 链接外部库

### 使用 `#[link]` 属性

```rust
// 链接 libm 数学库
#[link(name = "m")]
extern "C" {
    fn sqrt(x: f64) -> f64;
    fn pow(base: f64, exp: f64) -> f64;
}

fn main() {
    unsafe {
        println!("sqrt(16) = {}", sqrt(16.0));
        println!("2^10 = {}", pow(2.0, 10.0));
    }
}
```

### 在 build.rs 中配置

```rust
// build.rs
fn main() {
    println!("cargo:rustc-link-lib=ssl");
    println!("cargo:rustc-link-lib=crypto");
}
```

---

## 📦 常见 C 类型映射

| C 类型 | Rust 类型 | 说明 |
|--------|-----------|------|
| `int` | `c_int` | 通常是 i32 |
| `long` | `c_long` | 平台相关 |
| `char` | `c_char` | i8 或 u8 |
| `void*` | `*mut c_void` | 通用指针 |
| `const char*` | `*const c_char` | C 字符串 |
| `size_t` | `usize` | 无符号大小 |

```rust
use std::ffi::{c_int, c_char, c_void, CStr, CString};
```

---

## 📝 处理 C 字符串

### C 字符串 vs Rust 字符串

- **C**: `\0` 结尾的字节数组
- **Rust**: UTF-8 带长度，无 `\0`

### Rust → C

```rust
use std::ffi::CString;

fn main() {
    // 创建 C 兼容字符串
    let rust_str = "Hello, C!";
    let c_string = CString::new(rust_str).unwrap();
    
    // 获取原始指针传给 C
    let ptr: *const i8 = c_string.as_ptr();
    
    unsafe {
        some_c_function(ptr);
    }
    // c_string 在这里仍然存活，ptr 有效
}
```

### C → Rust

```rust
use std::ffi::CStr;

unsafe fn process_c_string(ptr: *const i8) {
    if ptr.is_null() {
        return;
    }
    
    // 从 C 指针创建 CStr（不获取所有权）
    let c_str = CStr::from_ptr(ptr);
    
    // 转换为 Rust &str
    match c_str.to_str() {
        Ok(s) => println!("Got: {}", s),
        Err(_) => println!("Invalid UTF-8"),
    }
}
```

---

## 🔄 让 C 调用 Rust

### 导出 Rust 函数给 C

```rust
// 使用 C 调用约定
#[no_mangle]  // 防止名称修饰
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}

#[no_mangle]
pub extern "C" fn rust_hello() {
    println!("Hello from Rust!");
}
```

### 编译为动态库

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib"]  # 生成 .so / .dylib / .dll
```

### 从 C 调用

```c
// main.c
extern int rust_add(int a, int b);
extern void rust_hello();

int main() {
    rust_hello();
    printf("1 + 2 = %d\n", rust_add(1, 2));
    return 0;
}
```

---

## 🛡️ 安全封装模式

**原则：用 safe Rust API 封装 unsafe FFI**

```rust
mod ffi {
    use std::ffi::{c_int, c_char, CStr};
    
    extern "C" {
        fn c_get_version() -> *const c_char;
        fn c_compute(input: c_int) -> c_int;
    }
    
    // 安全封装
    pub fn get_version() -> Option<String> {
        unsafe {
            let ptr = c_get_version();
            if ptr.is_null() {
                return None;
            }
            CStr::from_ptr(ptr)
                .to_str()
                .ok()
                .map(|s| s.to_owned())
        }
    }
    
    pub fn compute(input: i32) -> i32 {
        unsafe { c_compute(input as c_int) as i32 }
    }
}

// 外部用户只用 safe API
fn main() {
    if let Some(v) = ffi::get_version() {
        println!("Version: {}", v);
    }
    println!("Result: {}", ffi::compute(42));
}
```

---

## 🔨 实战：调用 libc

```rust
use std::ffi::{c_int, c_char, CString};

extern "C" {
    fn getpid() -> c_int;
    fn getenv(name: *const c_char) -> *const c_char;
}

fn safe_getenv(key: &str) -> Option<String> {
    let c_key = CString::new(key).ok()?;
    unsafe {
        let ptr = getenv(c_key.as_ptr());
        if ptr.is_null() {
            None
        } else {
            std::ffi::CStr::from_ptr(ptr)
                .to_str()
                .ok()
                .map(|s| s.to_owned())
        }
    }
}

fn main() {
    unsafe {
        println!("PID: {}", getpid());
    }
    
    if let Some(home) = safe_getenv("HOME") {
        println!("HOME: {}", home);
    }
}
```

---

## 🔧 bindgen：自动生成绑定

手写 FFI 声明容易出错，用 **bindgen** 自动生成：

```toml
# Cargo.toml
[build-dependencies]
bindgen = "0.69"
```

```rust
// build.rs
fn main() {
    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .generate()
        .expect("Unable to generate bindings");
    
    bindings
        .write_to_file("src/bindings.rs")
        .expect("Couldn't write bindings");
}
```

---

## ⚠️ 常见陷阱

### 1. 字符串生命周期

```rust
// ❌ 错误：CString 被立即释放
let ptr = CString::new("hello").unwrap().as_ptr();
unsafe { use_ptr(ptr); }  // 悬垂指针！

// ✅ 正确：保持 CString 存活
let s = CString::new("hello").unwrap();
unsafe { use_ptr(s.as_ptr()); }
```

### 2. 内存所有权

```rust
// C 分配的内存，谁负责释放？
extern "C" {
    fn c_create() -> *mut Data;
    fn c_destroy(ptr: *mut Data);  // 必须配对！
}
```

### 3. 对齐和布局

```rust
// 确保与 C 布局一致
#[repr(C)]
struct Point {
    x: f64,
    y: f64,
}
```

---

## 💡 总结

| 场景 | 工具 |
|------|------|
| Rust 调 C | `extern "C" { }` + `unsafe` |
| C 调 Rust | `#[no_mangle] extern "C" fn` |
| C 字符串 | `CString` / `CStr` |
| 自动生成 | bindgen |
| 安全封装 | safe wrapper 函数 |

FFI 是 Rust 与外部世界的桥梁，也是 unsafe 最常见的合法用途！

---

*课程日期：2026-02-17*
