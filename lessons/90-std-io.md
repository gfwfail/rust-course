# 第 90 课：std::io — 输入输出的艺术

> 日期：2026-02-24

## 为什么讲这个？

几乎所有程序都要和外部世界打交道 — 读文件、写日志、网络通信。在 PHP/Node 里，IO 操作很"隐形"，你直接 `file_get_contents()` 或 `fs.readFileSync()` 就完事了。

但 Rust 的 `std::io` 设计得非常精妙：
- **零成本抽象** — trait 定义接口，具体类型实现细节
- **统一的 IO trait** — 文件、网络、内存缓冲区都用同一套 API
- **细粒度的错误处理** — 每种错误都有对应的类型

---

## 核心 trait：Read 和 Write

这是 Rust IO 的两大基石：

```rust
use std::io::{Read, Write};

// Read trait 的核心方法
pub trait Read {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize>;
    
    // 还有很多默认实现的便捷方法
    fn read_to_end(&mut self, buf: &mut Vec<u8>) -> io::Result<usize>;
    fn read_to_string(&mut self, buf: &mut String) -> io::Result<usize>;
    fn read_exact(&mut self, buf: &mut [u8]) -> io::Result<()>;
}

// Write trait 的核心方法
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize>;
    fn flush(&mut self) -> io::Result<()>;
    
    // 便捷方法
    fn write_all(&mut self, buf: &[u8]) -> io::Result<()>;
}
```

**类比 PHP：**
- `Read` ≈ `fread()` / `stream_get_contents()`
- `Write` ≈ `fwrite()`

**为什么要 `&mut self`？**
因为读写会改变内部状态（比如文件指针位置）。

---

## 实战：读取文件

```rust
use std::fs::File;
use std::io::{self, Read};

fn main() -> io::Result<()> {
    // 方式1：一次性读完（小文件）
    let mut file = File::open("hello.txt")?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    println!("{contents}");
    
    // 方式2：分块读取（大文件）
    let mut file = File::open("big_file.bin")?;
    let mut buffer = [0u8; 1024]; // 1KB 缓冲区
    
    loop {
        let bytes_read = file.read(&mut buffer)?;
        if bytes_read == 0 {
            break; // EOF
        }
        // 处理 buffer[..bytes_read]
        println!("读取了 {bytes_read} 字节");
    }
    
    Ok(())
}
```

**关键点：**
- `read()` 返回读取的字节数，0 表示 EOF
- 缓冲区由调用者提供 — Rust 风格，避免隐式堆分配

---

## BufRead — 带缓冲的读取

逐行读取是常见需求，但裸 `Read` 没有"行"的概念：

```rust
use std::fs::File;
use std::io::{self, BufRead, BufReader};

fn main() -> io::Result<()> {
    let file = File::open("log.txt")?;
    let reader = BufReader::new(file); // 包一层缓冲
    
    // 方式1：迭代每一行
    for line in reader.lines() {
        let line = line?; // lines() 返回 Result
        println!("{line}");
    }
    
    // 方式2：读取直到某个分隔符
    let file = File::open("data.csv")?;
    let mut reader = BufReader::new(file);
    let mut field = Vec::new();
    reader.read_until(b',', &mut field)?;
    
    Ok(())
}
```

**为什么需要 BufReader？**
- 每次 `read()` 都是系统调用，开销大
- BufReader 在内存里缓存数据，减少系统调用次数
- `lines()` 需要 `BufRead` trait

**类比：**
- `BufReader` ≈ PHP 的 `fgets()` 内部缓冲
- `lines()` ≈ `file()` 函数按行读取

---

## BufWriter — 带缓冲的写入

写入同理：

```rust
use std::fs::File;
use std::io::{self, BufWriter, Write};

fn main() -> io::Result<()> {
    let file = File::create("output.txt")?;
    let mut writer = BufWriter::new(file);
    
    // 多次小写入，BufWriter 会攒起来
    for i in 0..1000 {
        writeln!(writer, "Line {i}")?;
    }
    
    // 显式 flush，确保数据写入
    writer.flush()?;
    
    // 或者 drop 时自动 flush
    Ok(())
}
```

**关键：** `BufWriter` drop 时会 flush，但如果 flush 失败，错误会被忽略！所以重要数据要显式 `flush()`。

---

## io::Cursor — 内存当文件用

有时候你想在内存里模拟文件操作：

```rust
use std::io::{Cursor, Read, Write, Seek, SeekFrom};

fn main() {
    // 用 Vec<u8> 模拟文件
    let mut cursor = Cursor::new(Vec::new());
    
    // 写入数据
    cursor.write_all(b"Hello, ").unwrap();
    cursor.write_all(b"Rust!").unwrap();
    
    // 回到开头
    cursor.seek(SeekFrom::Start(0)).unwrap();
    
    // 读取
    let mut output = String::new();
    cursor.read_to_string(&mut output).unwrap();
    println!("{output}"); // Hello, Rust!
    
    // 获取底层数据
    let data: Vec<u8> = cursor.into_inner();
}
```

**用途：**
- 单元测试不用真建文件
- 内存中构建二进制数据
- 实现需要 `Read`/`Write` 的接口但数据在内存里

---

## 标准输入输出

```rust
use std::io::{self, Write, BufRead};

fn main() -> io::Result<()> {
    // 标准输出 (println! 的底层)
    let stdout = io::stdout();
    let mut handle = stdout.lock(); // 锁住避免竞争
    writeln!(handle, "Hello from stdout")?;
    
    // 标准错误
    let stderr = io::stderr();
    let mut handle = stderr.lock();
    writeln!(handle, "Error message")?;
    
    // 标准输入
    let stdin = io::stdin();
    let handle = stdin.lock();
    
    print!("Enter your name: ");
    io::stdout().flush()?; // print! 不自动 flush
    
    for line in handle.lines() {
        let line = line?;
        println!("You entered: {line}");
        break;
    }
    
    Ok(())
}
```

**注意：** `print!` 不会自动 flush，如果要立即显示要手动 flush！

---

## io::copy — 流式复制

```rust
use std::fs::File;
use std::io;

fn main() -> io::Result<()> {
    let mut source = File::open("input.txt")?;
    let mut dest = File::create("output.txt")?;
    
    // 高效复制，不需要把整个文件读入内存
    let bytes_copied = io::copy(&mut source, &mut dest)?;
    println!("复制了 {bytes_copied} 字节");
    
    Ok(())
}
```

`io::copy` 内部用缓冲区分块复制，适合大文件。

---

## 错误处理：io::Error

```rust
use std::io::{self, ErrorKind};

fn read_config() -> io::Result<String> {
    match std::fs::read_to_string("config.toml") {
        Ok(content) => Ok(content),
        Err(e) => {
            // 根据错误类型处理
            match e.kind() {
                ErrorKind::NotFound => {
                    println!("配置文件不存在，使用默认配置");
                    Ok(String::from("[default]"))
                }
                ErrorKind::PermissionDenied => {
                    Err(io::Error::new(
                        ErrorKind::PermissionDenied,
                        "没有读取配置文件的权限"
                    ))
                }
                _ => Err(e),
            }
        }
    }
}
```

**常见 ErrorKind：**
- `NotFound` — 文件不存在
- `PermissionDenied` — 权限不足
- `ConnectionRefused` — 连接被拒
- `UnexpectedEof` — 意外的 EOF
- `WouldBlock` — 非阻塞操作会阻塞

---

## 💡 要点总结

1. **Read + Write** — IO 的核心抽象，文件/网络/内存都实现它们
2. **BufReader/BufWriter** — 减少系统调用，提供便捷方法
3. **Cursor** — 内存模拟文件，测试利器
4. **io::copy** — 流式复制，内存友好
5. **显式 flush** — 重要数据别靠 drop

---

## 下节预告

下节课我们讲 `std::fs` — 文件系统操作 🔧
