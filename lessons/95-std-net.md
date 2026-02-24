# 第 95 课：std::net — 网络编程基础

> 日期：2026-02-24

Rust 标准库的网络模块 `std::net` 提供**同步阻塞**的 TCP/UDP 网络原语。

---

## 📡 模块概览

```rust
use std::net::{
    // IP 地址
    IpAddr, Ipv4Addr, Ipv6Addr,
    // Socket 地址
    SocketAddr, SocketAddrV4, SocketAddrV6,
    // TCP
    TcpListener, TcpStream,
    // UDP
    UdpSocket,
    // 地址解析
    ToSocketAddrs,
};
```

> 💡 注意：`std::net` 是**同步阻塞**的。生产环境的 Web 服务通常用 Tokio 的异步版本 `tokio::net`。但理解标准库是基础！

---

## 🌐 IP 地址

```rust
use std::net::{IpAddr, Ipv4Addr, Ipv6Addr};

fn main() {
    // IPv4 地址
    let v4 = Ipv4Addr::new(127, 0, 0, 1);
    println!("IPv4: {}", v4);           // 127.0.0.1
    println!("是回环? {}", v4.is_loopback());  // true
    
    // IPv6 地址
    let v6 = Ipv6Addr::new(0, 0, 0, 0, 0, 0, 0, 1);
    println!("IPv6: {}", v6);           // ::1
    
    // 通用 IpAddr（可以是 v4 或 v6）
    let ip: IpAddr = "192.168.1.1".parse().unwrap();
    
    match ip {
        IpAddr::V4(v4) => println!("IPv4: {}", v4),
        IpAddr::V6(v6) => println!("IPv6: {}", v6),
    }
    
    // 常用方法
    println!("是私有地址? {}", Ipv4Addr::new(192, 168, 1, 1).is_private());
    println!("是广播? {}", Ipv4Addr::new(255, 255, 255, 255).is_broadcast());
}
```

---

## 📍 Socket 地址

Socket 地址 = IP + 端口：

```rust
use std::net::{SocketAddr, SocketAddrV4, Ipv4Addr};

fn main() {
    // 方式 1：直接解析字符串
    let addr: SocketAddr = "127.0.0.1:8080".parse().unwrap();
    println!("地址: {}, 端口: {}", addr.ip(), addr.port());
    
    // 方式 2：构造
    let addr = SocketAddrV4::new(Ipv4Addr::new(127, 0, 0, 1), 3000);
    println!("构造的地址: {}", addr);
    
    // 方式 3：From tuple（最简洁）
    let addr = SocketAddr::from(([127, 0, 0, 1], 8080));
    println!("From tuple: {}", addr);
}
```

---

## 🔌 TCP Server（服务端）

```rust
use std::net::{TcpListener, TcpStream};
use std::io::{Read, Write};

fn handle_client(mut stream: TcpStream) {
    let mut buffer = [0; 1024];
    
    // 读取请求
    let n = stream.read(&mut buffer).unwrap();
    let request = String::from_utf8_lossy(&buffer[..n]);
    println!("收到请求:\n{}", request);
    
    // 发送响应
    let response = "HTTP/1.1 200 OK\r\n\r\nHello from Rust!";
    stream.write_all(response.as_bytes()).unwrap();
}

fn main() {
    // 监听端口
    let listener = TcpListener::bind("127.0.0.1:3000").unwrap();
    println!("🚀 服务器启动在 http://127.0.0.1:3000");
    
    // 接受连接（阻塞）
    for stream in listener.incoming() {
        match stream {
            Ok(stream) => {
                println!("新连接: {}", stream.peer_addr().unwrap());
                handle_client(stream);
            }
            Err(e) => eprintln!("连接失败: {}", e),
        }
    }
}
```

类比 PHP：
```php
// PHP 的 socket_create + socket_bind + socket_listen
$socket = stream_socket_server("tcp://127.0.0.1:3000");
while ($conn = stream_socket_accept($socket)) {
    fwrite($conn, "Hello");
    fclose($conn);
}
```

---

## 📱 TCP Client（客户端）

```rust
use std::net::TcpStream;
use std::io::{Read, Write};
use std::time::Duration;

fn main() -> std::io::Result<()> {
    // 连接服务器
    let mut stream = TcpStream::connect("127.0.0.1:3000")?;
    
    // 设置超时
    stream.set_read_timeout(Some(Duration::from_secs(5)))?;
    stream.set_write_timeout(Some(Duration::from_secs(5)))?;
    
    // 发送 HTTP 请求
    let request = "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n";
    stream.write_all(request.as_bytes())?;
    
    // 读取响应
    let mut response = String::new();
    stream.read_to_string(&mut response)?;
    println!("响应:\n{}", response);
    
    Ok(())
}
```

---

## 📨 UDP（无连接）

```rust
use std::net::UdpSocket;

// UDP 服务端
fn udp_server() -> std::io::Result<()> {
    let socket = UdpSocket::bind("127.0.0.1:3000")?;
    println!("UDP 服务器启动");
    
    let mut buf = [0; 1024];
    loop {
        let (len, src) = socket.recv_from(&mut buf)?;
        println!("收到来自 {} 的消息: {}", src, 
                 String::from_utf8_lossy(&buf[..len]));
        
        // 回复
        socket.send_to(b"ACK", src)?;
    }
}

// UDP 客户端
fn udp_client() -> std::io::Result<()> {
    let socket = UdpSocket::bind("127.0.0.1:0")?;  // 0 = 随机端口
    socket.connect("127.0.0.1:3000")?;  // "连接"（只是设置默认目标）
    
    socket.send(b"Hello UDP!")?;
    
    let mut buf = [0; 1024];
    let len = socket.recv(&mut buf)?;
    println!("收到: {}", String::from_utf8_lossy(&buf[..len]));
    
    Ok(())
}
```

### TCP vs UDP

| 特性 | TCP | UDP |
|------|-----|-----|
| 连接 | 需要建立连接 | 无连接 |
| 可靠性 | 保证送达、顺序 | 不保证 |
| 速度 | 较慢 | 更快 |
| 用途 | HTTP, 数据库 | DNS, 游戏, 视频 |

---

## 🔍 ToSocketAddrs trait

这个 trait 让你可以用多种方式指定地址：

```rust
use std::net::{TcpStream, ToSocketAddrs};

fn connect<A: ToSocketAddrs>(addr: A) -> std::io::Result<TcpStream> {
    TcpStream::connect(addr)
}

fn main() -> std::io::Result<()> {
    // 这些都可以！
    connect("127.0.0.1:80")?;              // &str
    connect(("127.0.0.1", 80))?;           // (&str, u16)
    connect(([127, 0, 0, 1], 80))?;        // ([u8; 4], u16)
    
    // 甚至支持 DNS 解析
    connect("example.com:80")?;
    
    Ok(())
}
```

---

## ⚡ 多线程 TCP Server

单线程服务器一次只能处理一个连接。加上多线程：

```rust
use std::net::{TcpListener, TcpStream};
use std::io::{Read, Write};
use std::thread;

fn handle_client(mut stream: TcpStream) {
    let peer = stream.peer_addr().unwrap();
    println!("[{}] 新连接", peer);
    
    let mut buffer = [0; 1024];
    loop {
        match stream.read(&mut buffer) {
            Ok(0) => {
                println!("[{}] 连接关闭", peer);
                break;
            }
            Ok(n) => {
                // Echo 服务器：收到什么就回什么
                stream.write_all(&buffer[..n]).unwrap();
            }
            Err(e) => {
                eprintln!("[{}] 读取错误: {}", peer, e);
                break;
            }
        }
    }
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:3000")?;
    println!("🚀 多线程服务器启动");
    
    for stream in listener.incoming() {
        let stream = stream?;
        
        // 每个连接一个线程
        thread::spawn(|| {
            handle_client(stream);
        });
    }
    
    Ok(())
}
```

---

## 🎯 实战：简单的 Key-Value 服务器

```rust
use std::net::{TcpListener, TcpStream};
use std::io::{BufRead, BufReader, Write};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::thread;

type Db = Arc<Mutex<HashMap<String, String>>>;

fn handle_client(mut stream: TcpStream, db: Db) {
    let reader = BufReader::new(stream.try_clone().unwrap());
    
    for line in reader.lines() {
        let line = match line {
            Ok(l) => l,
            Err(_) => break,
        };
        
        let parts: Vec<&str> = line.trim().split_whitespace().collect();
        
        let response = match parts.as_slice() {
            ["GET", key] => {
                let db = db.lock().unwrap();
                match db.get(*key) {
                    Some(v) => format!("VALUE: {}\n", v),
                    None => "NOT_FOUND\n".to_string(),
                }
            }
            ["SET", key, value] => {
                let mut db = db.lock().unwrap();
                db.insert(key.to_string(), value.to_string());
                "OK\n".to_string()
            }
            ["DEL", key] => {
                let mut db = db.lock().unwrap();
                match db.remove(*key) {
                    Some(_) => "DELETED\n".to_string(),
                    None => "NOT_FOUND\n".to_string(),
                }
            }
            _ => "ERROR: Unknown command\n".to_string(),
        };
        
        stream.write_all(response.as_bytes()).unwrap();
    }
}

fn main() -> std::io::Result<()> {
    let db: Db = Arc::new(Mutex::new(HashMap::new()));
    let listener = TcpListener::bind("127.0.0.1:6379")?;
    
    println!("🗄️ Mini Redis 启动在 127.0.0.1:6379");
    println!("命令: GET key | SET key value | DEL key");
    
    for stream in listener.incoming() {
        let stream = stream?;
        let db = Arc::clone(&db);
        
        thread::spawn(move || {
            handle_client(stream, db);
        });
    }
    
    Ok(())
}
```

用 telnet 测试：
```bash
$ telnet 127.0.0.1 6379
SET name rust
OK
GET name
VALUE: rust
```

---

## 📝 要点总结

| 类型 | 用途 |
|------|------|
| `IpAddr` | IPv4/IPv6 地址 |
| `SocketAddr` | IP + 端口 |
| `TcpListener` | TCP 服务器监听 |
| `TcpStream` | TCP 连接（双向读写） |
| `UdpSocket` | UDP 通信 |
| `ToSocketAddrs` | 地址转换 trait |

**关键点：**
1. `std::net` 是**同步阻塞**的
2. 生产环境用 `tokio::net`（异步）
3. TCP 需要连接，UDP 不需要
4. 多连接需要多线程或异步

---

*下节课：std::time — 时间与持续时间*
