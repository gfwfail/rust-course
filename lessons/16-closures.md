# 第16课：闭包 Closures

闭包就是**匿名函数**，可以保存在变量里、当参数传递，而且能**捕获**环境中的变量。

## 基本语法

```rust
// 普通函数
fn add_one(x: i32) -> i32 {
    x + 1
}

// 闭包
let add_one = |x: i32| -> i32 { x + 1 };

// 简写版（类型推断 + 单表达式）
let add_one = |x| x + 1;

// 无参数
let say_hi = || println!("Hi!");

// 多行
let complex = |x| {
    let y = x * 2;
    y + 1
};
```

**核心区别**：用 `||` 而不是 `()`

## 捕获环境变量

这是闭包比普通函数厉害的地方！

```rust
fn main() {
    let factor = 10;
    
    // 闭包可以用外面的 factor！
    let multiply = |x| x * factor;
    
    println!("{}", multiply(5)); // 50
    
    // 普通函数做不到这个
    // fn multiply_fn(x: i32) -> i32 { x * factor } // 错误！
}
```

## 三种捕获方式

闭包根据使用方式自动选择如何捕获：

```rust
fn main() {
    let s = String::from("hello");
    
    // 1️⃣ 借用（默认）- 实现 Fn
    let borrow = || println!("{}", s);
    borrow();
    println!("{}", s); // s 还能用
    
    // 2️⃣ 可变借用 - 实现 FnMut
    let mut count = 0;
    let mut increment = || count += 1;
    increment();
    increment();
    println!("{}", count); // 2
    
    // 3️⃣ 移动所有权 - 实现 FnOnce
    let s2 = String::from("world");
    let consume = move || println!("{}", s2);
    consume();
    // println!("{}", s2); // 错误！s2 已移入闭包
}
```

## 三个 Trait：Fn、FnMut、FnOnce

| Trait | 捕获方式 | 能调用几次 |
|-------|---------|-----------|
| `FnOnce` | 移动（消耗） | 至少1次 |
| `FnMut` | 可变借用 | 多次 |
| `Fn` | 不可变借用 | 多次 |

```rust
fn call_fn<F: Fn()>(f: F) {
    f();
    f(); // 可以多次调用
}

fn call_fn_once<F: FnOnce()>(f: F) {
    f(); // 只能调一次
    // f(); // 错误！
}
```

## 配合迭代器

闭包最常用的场景！

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    
    // map：对每个元素应用闭包
    let doubled: Vec<i32> = numbers
        .iter()
        .map(|x| x * 2)
        .collect();
    
    // filter：保留满足条件的
    let evens: Vec<&i32> = numbers
        .iter()
        .filter(|x| *x % 2 == 0)
        .collect();
    
    // 捕获外部变量
    let threshold = 3;
    let above: Vec<&i32> = numbers
        .iter()
        .filter(|x| **x > threshold)
        .collect();
}
```

## move 关键字

强制闭包获取所有权：

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    
    // 线程可能比 main 活得久，必须 move
    let handle = thread::spawn(move || {
        println!("{:?}", data);
    });
    
    // println!("{:?}", data); // 错误！data 已移入线程
    handle.join().unwrap();
}
```

## 返回闭包

因为闭包大小不固定，返回时要用 `Box`：

```rust
fn make_adder(n: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x + n)
}

fn main() {
    let add_5 = make_adder(5);
    println!("{}", add_5(10)); // 15
}
```

---

**下节课**：Box 堆分配
