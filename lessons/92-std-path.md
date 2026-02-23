# 第 92 课：std::path — 路径操作的艺术

> 日期：2026-02-24  
> 主题：标准库路径处理模块

---

## 🎯 为什么需要 Path？

你可能会问：路径不就是字符串吗？为什么要专门搞个类型？

```rust
// ❌ 这样写跨平台会出问题
let path = "/home/user/file.txt";  // Unix 风格
let path = "C:\\Users\\file.txt";  // Windows 风格

// ✅ 用 Path，Rust 帮你处理差异
use std::path::Path;
let path = Path::new("/home/user/file.txt");
```

**三个核心原因：**
1. **跨平台**：自动处理 `/` vs `\` 的差异
2. **类型安全**：编译时就能发现路径操作错误
3. **丰富 API**：提取文件名、扩展名、父目录等

---

## 🔧 Path vs PathBuf

这对关系就像 `str` vs `String`：

```rust
use std::path::{Path, PathBuf};

// Path = 借用，不可变，类似 &str
let path: &Path = Path::new("/home/user/file.txt");

// PathBuf = 拥有所有权，可变，类似 String
let mut path_buf: PathBuf = PathBuf::from("/home/user");
path_buf.push("file.txt");  // 可以修改！
```

**PHP/Laravel 对比：**
```php
// PHP 里路径就是字符串
$path = '/home/user';
$fullPath = $path . '/file.txt';

// Rust 更安全
let mut path = PathBuf::from("/home/user");
path.push("file.txt");  // 自动处理分隔符
```

---

## 📁 创建路径

```rust
use std::path::{Path, PathBuf};

// 方式1：Path::new（借用）
let p1: &Path = Path::new("foo/bar.txt");

// 方式2：PathBuf::from（拥有）
let p2: PathBuf = PathBuf::from("foo/bar.txt");

// 方式3：字符串转换
let s = String::from("foo/bar.txt");
let p3: PathBuf = s.into();
let p4: &Path = Path::new(&s);

// 方式4：从组件构建
let p5: PathBuf = ["home", "user", "docs"].iter().collect();
println!("{}", p5.display());  // home/user/docs
```

---

## 🔍 路径分析 API

```rust
use std::path::Path;

let path = Path::new("/home/user/documents/report.pdf");

// 获取文件名
println!("{:?}", path.file_name());     
// Some("report.pdf")

// 获取扩展名
println!("{:?}", path.extension());     
// Some("pdf")

// 获取不带扩展名的文件名
println!("{:?}", path.file_stem());     
// Some("report")

// 获取父目录
println!("{:?}", path.parent());        
// Some("/home/user/documents")

// 是否绝对路径
println!("{}", path.is_absolute());     
// true

// 是否相对路径
println!("{}", path.is_relative());     
// false
```

---

## 🛠️ PathBuf 修改操作

```rust
use std::path::PathBuf;

let mut path = PathBuf::from("/home/user");

// push - 追加组件
path.push("docs");
path.push("file.txt");
println!("{}", path.display());  // /home/user/docs/file.txt

// pop - 移除最后一个组件
path.pop();
println!("{}", path.display());  // /home/user/docs

// set_file_name - 替换文件名
path.push("old.txt");
path.set_file_name("new.txt");
println!("{}", path.display());  // /home/user/docs/new.txt

// set_extension - 设置扩展名
path.set_extension("md");
println!("{}", path.display());  // /home/user/docs/new.md

// 添加扩展名（不替换）
path.set_extension("md.bak");
println!("{}", path.display());  // /home/user/docs/new.md.bak
```

---

## 🔗 路径拼接

```rust
use std::path::{Path, PathBuf};

// 方式1：push（修改原 PathBuf）
let mut path = PathBuf::from("/home");
path.push("user");  // 注意：push 会自动加分隔符

// 方式2：join（返回新 PathBuf，不修改原路径）
let base = Path::new("/home/user");
let full = base.join("docs").join("file.txt");
println!("{}", full.display());  // /home/user/docs/file.txt

// ⚠️ 注意：join 绝对路径会替换
let weird = base.join("/etc/passwd");
println!("{}", weird.display());  // /etc/passwd（不是你想的那样！）
```

**重要提醒：** `join` 一个绝对路径会直接替换，不是拼接！

---

## 🔄 路径与字符串转换

```rust
use std::path::{Path, PathBuf};
use std::ffi::OsStr;

let path = Path::new("/home/user/文档/报告.pdf");

// Path → &str（可能失败，因为路径不一定是有效 UTF-8）
if let Some(s) = path.to_str() {
    println!("字符串: {}", s);
}

// Path → OsStr（总是成功）
let os_str: &OsStr = path.as_os_str();

// Path → String（会用 � 替换非 UTF-8 字符）
let string: String = path.to_string_lossy().into_owned();

// &str → Path
let path: &Path = Path::new("/home/user");

// String → PathBuf
let path_buf: PathBuf = String::from("/home/user").into();
```

---

## 🌍 跨平台实战

```rust
use std::path::{Path, PathBuf, MAIN_SEPARATOR};

// MAIN_SEPARATOR：当前平台的路径分隔符
println!("分隔符: {}", MAIN_SEPARATOR);  
// Unix: /    Windows: \

// 规范化路径（处理 . 和 ..）
fn normalize_path(path: &Path) -> PathBuf {
    let mut result = PathBuf::new();
    
    for component in path.components() {
        use std::path::Component;
        match component {
            Component::ParentDir => { result.pop(); }
            Component::CurDir => {}  // 忽略 .
            c => result.push(c),
        }
    }
    result
}

let messy = Path::new("/home/user/../user/./docs/../docs/file.txt");
let clean = normalize_path(messy);
println!("{}", clean.display());  // /home/user/docs/file.txt
```

---

## 🔬 遍历路径组件

```rust
use std::path::Path;

let path = Path::new("/home/user/docs/file.txt");

// components() - 分解为组件
for comp in path.components() {
    println!("{:?}", comp);
}
// 输出：
// RootDir
// Normal("home")
// Normal("user")
// Normal("docs")
// Normal("file.txt")

// ancestors() - 所有祖先路径
for ancestor in path.ancestors() {
    println!("{}", ancestor.display());
}
// 输出：
// /home/user/docs/file.txt
// /home/user/docs
// /home/user
// /home
// /
```

---

## 💡 实战：安全路径拼接

防止路径遍历攻击（Laravel 里常见的安全问题）：

```rust
use std::path::{Path, PathBuf};

fn safe_join(base: &Path, user_input: &str) -> Option<PathBuf> {
    let requested = Path::new(user_input);
    
    // 拒绝绝对路径
    if requested.is_absolute() {
        return None;
    }
    
    // 拒绝包含 .. 的路径
    for component in requested.components() {
        if matches!(component, std::path::Component::ParentDir) {
            return None;
        }
    }
    
    let full = base.join(requested);
    
    // 确保结果在 base 目录下
    if full.starts_with(base) {
        Some(full)
    } else {
        None
    }
}

// 测试
let base = Path::new("/var/uploads");
assert!(safe_join(base, "user/file.txt").is_some());
assert!(safe_join(base, "../etc/passwd").is_none());  // 拒绝！
assert!(safe_join(base, "/etc/passwd").is_none());    // 拒绝！
```

---

## 📝 小结

| 类型 | 所有权 | 可变性 | 类似于 |
|------|--------|--------|--------|
| `Path` | 借用 | 不可变 | `str` |
| `PathBuf` | 拥有 | 可变 | `String` |

**关键 API：**
- 创建：`Path::new()`, `PathBuf::from()`
- 分析：`file_name()`, `extension()`, `parent()`
- 修改：`push()`, `pop()`, `set_extension()`
- 拼接：`join()`（注意绝对路径会替换！）
- 遍历：`components()`, `ancestors()`

---

## 🔗 相关资源

- [std::path 官方文档](https://doc.rust-lang.org/std/path/index.html)
- [Path 结构体](https://doc.rust-lang.org/std/path/struct.Path.html)
- [PathBuf 结构体](https://doc.rust-lang.org/std/path/struct.PathBuf.html)

---

*下节课：std::env — 环境变量与程序参数*
