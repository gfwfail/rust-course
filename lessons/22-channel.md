# 第22课：Channel 消息传递

Channel 就像一根管道：一头发送数据，一头接收数据。

Rust 的口号：「不要通过共享内存来通信，而要通过通信来共享内存」

## 创建 Channel

```rust
use std::sync::mpsc;

fn main() {
    // mpsc = Multi-Producer, Single-Consumer
    let (tx, rx) = mpsc::channel();
    
    // tx: 发送端 (Transmitter)
    // rx: 接收端 (Receiver)
}
```

## 发送与接收

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    // 子线程发送数据
    thread::spawn(move || {
        let msg = String::from("你好，主线程！");
        tx.send(msg).unwrap();
        // 注意：msg 已经被移动，不能再用！
    });
    
    // 主线程接收数据
    let received = rx.recv().unwrap();
    println!("收到: {}", received);
}
```

## 所有权转移

`send()` 会移动所有权，这是 Rust 保证线程安全的关键！

```rust
thread::spawn(move || {
    let msg = String::from("hello");
    tx.send(msg).unwrap();
    
    // ❌ 编译错误！msg 已被移动
    // println!("发送了: {}", msg);
});
```

## recv() vs try_recv()

```rust
// 阻塞等待，直到收到数据
let msg = rx.recv().unwrap();

// 非阻塞，立即返回
match rx.try_recv() {
    Ok(msg) => println!("收到: {}", msg),
    Err(_) => println!("暂时没数据"),
}
```

## 发送多条消息

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    thread::spawn(move || {
        let msgs = vec!["第一条", "第二条", "第三条"];
        for msg in msgs {
            tx.send(msg).unwrap();
            thread::sleep(Duration::from_millis(500));
        }
    });
    
    // 把 rx 当迭代器用！
    for received in rx {
        println!("收到: {}", received);
    }
}
```

## 多个发送者

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    let tx2 = tx.clone();  // 克隆发送端！
    
    thread::spawn(move || {
        tx.send("线程1: 你好").unwrap();
    });
    
    thread::spawn(move || {
        tx2.send("线程2: 嗨").unwrap();
    });
    
    for received in rx {
        println!("{}", received);
    }
}
```

⚠️ **注意**：必须 drop 所有发送端，rx 迭代才会结束！

---

**下节课**：Mutex 互斥锁
