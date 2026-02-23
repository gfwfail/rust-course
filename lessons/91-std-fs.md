# 第 91 课：std::fs — 文件系统操作

> 日期：2026-02-24

## 为什么讲这个？

上节课我们学了 `std::io` 的读写抽象。但光有读写还不够 — 你得能创建文件、删除文件、遍历目录、获取文件信息。

PHP 里这些是一堆零散函数：`file_exists()`、`mkdir()`、`unlink()`、`scandir()`...

Rust 的 `std::fs` 把这些整合成一套完整的 API，而且错误处理更严谨。

---

## 基础操作：读写文件

先看最简单的一次性读写：

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    // 读取整个文件为 String
    let content = fs::read_to_string("hello.txt")?;
    println!("{content}");
    
    // 读取为字节
    let bytes = fs::read("image.png")?;
    println!("文件大小: {} bytes", bytes.len());
    
    // 写入文件（覆盖）
    fs::write("output.txt", "Hello, Rust!")?;
    
    // 写入字节
    fs::write("data.bin", &[0x00, 0x01, 0x02])?;
    
    Ok(())
}
```

**类比 PHP：**
- `fs::read_to_string()` ≈ `file_get_contents()`
- `fs::write()` ≈ `file_put_contents()`

**注意：** 这些函数适合小文件。大文件用上节课的 `BufReader`/`BufWriter`。

---

## File — 文件句柄

需要更精细控制时，用 `File`：

```rust
use std::fs::{File, OpenOptions};
use std::io::{Write, Read, Seek, SeekFrom};

fn main() -> std::io::Result<()> {
    // 只读打开
    let mut file = File::open("data.txt")?;
    let mut content = String::new();
    file.read_to_string(&mut content)?;
    
    // 创建/覆盖（只写）
    let mut file = File::create("output.txt")?;
    file.write_all(b"Hello")?;
    
    // 复杂场景用 OpenOptions
    let mut file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)      // 不存在则创建
        .append(true)      // 追加模式
        .open("log.txt")?;
    
    writeln!(file, "New log entry")?;
    
    // 定位
    file.seek(SeekFrom::Start(0))?;      // 回到开头
    file.seek(SeekFrom::End(-10))?;       // 倒数第10字节
    file.seek(SeekFrom::Current(5))?;     // 往前5字节
    
    Ok(())
}
```

**OpenOptions 常用组合：**
- 追加日志：`.append(true).create(true)`
- 读写已存在文件：`.read(true).write(true)`
- 截断覆盖：`.write(true).truncate(true)`

---

## 文件元信息

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    // 获取元信息
    let metadata = fs::metadata("file.txt")?;
    
    println!("是文件: {}", metadata.is_file());
    println!("是目录: {}", metadata.is_dir());
    println!("大小: {} bytes", metadata.len());
    println!("只读: {}", metadata.permissions().readonly());
    
    // 修改时间（如果平台支持）
    if let Ok(modified) = metadata.modified() {
        println!("修改时间: {:?}", modified);
    }
    
    // 不跟随符号链接
    let metadata = fs::symlink_metadata("link")?;
    println!("是符号链接: {}", metadata.is_symlink());
    
    Ok(())
}
```

**类比 PHP：**
- `metadata()` ≈ `stat()`
- `is_file()` ≈ `is_file()`
- `len()` ≈ `filesize()`

---

## 目录操作

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    // 创建单层目录
    fs::create_dir("new_folder")?;
    
    // 递归创建（含父目录）
    fs::create_dir_all("path/to/deep/folder")?;
    
    // 删除空目录
    fs::remove_dir("empty_folder")?;
    
    // 递归删除（危险！）
    fs::remove_dir_all("folder_with_contents")?;
    
    // 遍历目录
    for entry in fs::read_dir(".")? {
        let entry = entry?;
        let path = entry.path();
        let file_type = entry.file_type()?;
        
        if file_type.is_file() {
            println!("文件: {}", path.display());
        } else if file_type.is_dir() {
            println!("目录: {}", path.display());
        }
    }
    
    Ok(())
}
```

**类比 PHP：**
- `create_dir_all()` ≈ `mkdir($path, 0755, true)`
- `remove_dir_all()` ≈ 递归 `rmdir()` + `unlink()`
- `read_dir()` ≈ `scandir()` 或 `DirectoryIterator`

---

## 文件操作

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    // 删除文件
    fs::remove_file("unwanted.txt")?;
    
    // 重命名/移动
    fs::rename("old.txt", "new.txt")?;
    fs::rename("file.txt", "other_folder/file.txt")?;
    
    // 复制（不是移动）
    let bytes_copied = fs::copy("source.txt", "dest.txt")?;
    println!("复制了 {bytes_copied} 字节");
    
    // 硬链接
    fs::hard_link("original.txt", "link.txt")?;
    
    Ok(())
}
```

**注意：** `fs::copy()` 只复制内容，不复制权限。

---

## 权限管理

```rust
use std::fs::{self, Permissions};

fn main() -> std::io::Result<()> {
    // 读取权限
    let metadata = fs::metadata("file.txt")?;
    let permissions = metadata.permissions();
    println!("只读: {}", permissions.readonly());
    
    // 设置只读
    let mut perms = permissions.clone();
    perms.set_readonly(true);
    fs::set_permissions("file.txt", perms)?;
    
    // Unix 专属：设置模式
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        
        let perms = Permissions::from_mode(0o755);
        fs::set_permissions("script.sh", perms)?;
        
        let mode = fs::metadata("script.sh")?.permissions().mode();
        println!("权限: {:o}", mode);
    }
    
    Ok(())
}
```

**跨平台注意：** `readonly()` 是跨平台的，但精细的 Unix 权限需要 `#[cfg(unix)]`。

---

## 实战：递归遍历目录

标准库没有递归遍历，但自己写很简单：

```rust
use std::fs;
use std::path::Path;

fn visit_dirs(dir: &Path) -> std::io::Result<()> {
    if dir.is_dir() {
        for entry in fs::read_dir(dir)? {
            let entry = entry?;
            let path = entry.path();
            
            if path.is_dir() {
                visit_dirs(&path)?; // 递归
            } else {
                println!("文件: {}", path.display());
            }
        }
    }
    Ok(())
}

fn main() -> std::io::Result<()> {
    visit_dirs(Path::new("."))?;
    Ok(())
}
```

**生产环境推荐：** `walkdir` crate，处理了符号链接循环等边界情况。

---

## 临时文件

```rust
use std::env;
use std::fs::File;
use std::io::Write;

fn main() -> std::io::Result<()> {
    // 获取临时目录
    let tmp_dir = env::temp_dir();
    println!("临时目录: {}", tmp_dir.display());
    
    // 创建临时文件（简单版）
    let tmp_path = tmp_dir.join("my_temp_file.txt");
    let mut file = File::create(&tmp_path)?;
    writeln!(file, "临时数据")?;
    
    // 用完记得删除
    std::fs::remove_file(&tmp_path)?;
    
    Ok(())
}
```

**生产环境推荐：** `tempfile` crate，自动清理、防止竞争条件。

---

## 错误处理模式

```rust
use std::fs;
use std::io::ErrorKind;

fn ensure_dir(path: &str) -> std::io::Result<()> {
    match fs::create_dir(path) {
        Ok(()) => Ok(()),
        Err(e) if e.kind() == ErrorKind::AlreadyExists => {
            // 目录已存在，没问题
            Ok(())
        }
        Err(e) => Err(e),
    }
}

fn safe_delete(path: &str) -> std::io::Result<()> {
    match fs::remove_file(path) {
        Ok(()) => Ok(()),
        Err(e) if e.kind() == ErrorKind::NotFound => {
            // 文件不存在，也算成功
            Ok(())
        }
        Err(e) => Err(e),
    }
}
```

**常见 ErrorKind：**
- `NotFound` — 文件/目录不存在
- `AlreadyExists` — 已存在
- `PermissionDenied` — 权限不足
- `DirectoryNotEmpty` — 目录非空

---

## 💡 要点总结

| 场景 | 函数 |
|------|------|
| 快速读文件 | `fs::read_to_string()` / `fs::read()` |
| 快速写文件 | `fs::write()` |
| 精细控制 | `File::open()` / `OpenOptions` |
| 创建目录 | `fs::create_dir_all()` |
| 删除文件 | `fs::remove_file()` |
| 递归删除 | `fs::remove_dir_all()` ⚠️ 危险 |
| 遍历目录 | `fs::read_dir()` |
| 获取信息 | `fs::metadata()` |

---

## 下节预告

下节课我们讲 `std::path` — 路径处理的正确姿势 🛤️
