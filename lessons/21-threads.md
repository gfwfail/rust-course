# 第21课：线程基础

今天进入 Rust 最强大的领域之一：**无畏并发（Fearless Concurrency）**！

## 创建线程

```rust
use std::thread;
use std::time::Duration;

fn main() {
    // 启动一个新线程
    thread::spawn(|| {
        for i in 1..5 {
            println!("子线程：{}", i);
            thread::sleep(Duration::from_millis(100));
        }
    });

    // 主线程继续执行
    for i in 1..3 {
        println!("主线程：{}", i);
        thread::sleep(Duration::from_millis(150));
    }
}
```

⚠️ **注意**：主线程结束，子线程也会被强制终止！

## 等待线程完成：JoinHandle

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..5 {
            println!("子线程：{}", i);
        }
        "任务完成！"  // 线程可以返回值
    });

    // 等待子线程结束，获取返回值
    let result = handle.join().unwrap();
    println!("子线程返回：{}", result);
}
```

## move 闭包：转移所有权

线程需要数据？必须用 `move` 转移所有权：

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];

    // ❌ 不行：闭包借用 data，但主线程可能先结束
    // let handle = thread::spawn(|| {
    //     println!("{:?}", data);
    // });

    // ✅ 用 move 转移所有权给新线程
    let handle = thread::spawn(move || {
        println!("子线程拿到：{:?}", data);
    });

    // ❌ data 已被移动，不能再用
    // println!("{:?}", data);

    handle.join().unwrap();
}
```

## 多线程计算

```rust
use std::thread;

fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8];
    let mid = numbers.len() / 2;
    let (left, right) = numbers.split_at(mid);
    
    let left = left.to_vec();
    let right = right.to_vec();

    let handle1 = thread::spawn(move || {
        left.iter().sum::<i32>()
    });

    let handle2 = thread::spawn(move || {
        right.iter().sum::<i32>()
    });

    let sum1 = handle1.join().unwrap();
    let sum2 = handle2.join().unwrap();
    
    println!("总和：{}", sum1 + sum2);
}
```

## Rust 的承诺

编译通过的代码，不会有数据竞争！

```rust
let mut counter = 0;

// ❌ 多线程同时修改？编译器不允许！
// thread::spawn(move || { counter += 1; });
// counter += 1;  // 编译错误
```

---

**下节课**：Channel 消息传递
