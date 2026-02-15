# 🦀 Rust 学习小组课程笔记

> 授课时间：2026-02-14 ~ 2026-02-16
> 授课对象：Web 程序员背景（Laravel/PHP/JS）
> 教学风格：实战导向，对比其他语言，讲「为什么」而不只是「怎么写」

---

## 目录

### 基础篇
1. [Hello World 与基础概念](#第1课hello-world-与基础概念)
2. [变量与数据类型](#第2课变量与数据类型)
3. [所有权 Ownership](#第3课所有权-ownership)
4. [生命周期 Lifetime](#第4课生命周期-lifetime)
5. [错误处理](#第5课错误处理)
6. [Rust 设计模式](#第6课rust-设计模式)

### 进阶篇
7. [Rust 特殊语法](#第7课rust-特殊语法)
8. [Enum 实战场景](#第8课enum-实战场景)
9. [泛型 Generics](#第9课泛型-generics)
10. [Trait 入门](#第10课trait-入门)
11. [Trait 对象与 dyn](#第11课trait-对象与-dyn)

### 集合与迭代
12. [Vec 动态数组](#第12课vec-动态数组)
13. [String 深入](#第13课string-深入)
14. [HashMap](#第14课hashmap)
15. [迭代器 Iterators](#第15课迭代器-iterators)
16. [闭包 Closures](#第16课闭包-closures)

### 智能指针
17. [Box 堆分配](#第17课box-堆分配)
18. [Rc 引用计数](#第18课rc-引用计数)
19. [RefCell 内部可变性](#第19课refcell-内部可变性)
20. [Weak 与循环引用](#第20课weak-与循环引用)

### 并发编程
21. [线程基础](#第21课线程基础)
22. [Channel 消息传递](#第22课channel-消息传递)
23. [Mutex 互斥锁](#第23课mutex-互斥锁)
24. [RwLock 读写锁](#第24课rwlock-读写锁)
25. [Send 与 Sync](#第25课send-与-sync)
26. [原子类型 Atomics](#第26课原子类型-atomics)

### 异步编程
27. [async/await 入门](#第27课asyncawait-入门)

### Q&A 问答整理
- [Borrow 的好处](#qa-borrow-的好处)
- [多线程数据竞争怎么识别](#qa-多线程数据竞争怎么识别)
- [生命周期防住什么 Bug](#qa-生命周期防住什么-bug)
- [Rust vs Laravel 吞吐量](#qa-rust-vs-laravel-吞吐量)

---

## 第1课：Hello World 与基础概念

### Rust 是什么？

- **内存安全**：编译期捕获大部分内存错误，没有 GC
- **零成本抽象**：高级特性不牺牲性能
- **并发安全**：编译器防止数据竞争

### 安装

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 验证
rustc --version
cargo --version
```

### Hello World

```bash
cargo new hello_rust
cd hello_rust
cargo run
```

```rust
fn main() {
    println!("Hello, World!");
}
```

- `fn main()` — 程序入口函数
- `println!` — 宏（注意感叹号 `!`），不是普通函数
- 语句结尾加分号 `;`

---

## 第2课：变量与数据类型

### 默认不可变

```rust
let x = 5;
x = 6;  // ❌ 编译错误！
```

要可变，加 `mut`：

```rust
let mut x = 5;
x = 6;  // ✅ OK
```

**什么时候用 `mut`？**
- 计数器、累加器
- 循环里要修改的状态
- 构建一个东西（拼字符串、往 Vec 里 push）

### 基本数据类型

| 类型 | 说明 |
|------|------|
| `i32` | 默认整数类型 |
| `i64` / `u64` | 大整数 / 数据库 ID |
| `usize` | 数组索引、`.len()` |
| `f64` | 默认浮点类型 |
| `bool` | true / false |
| `char` | Unicode 字符 |

### Shadowing（变量遮蔽）

```rust
let user_id = "123";           // &str
let user_id: u32 = user_id.parse().unwrap();  // 遮蔽为 u32
```

**用途**：数据类型转换、让旧版本数据不可访问防止误用

---

## 第3课：所有权 Ownership

### 三条铁律

1. 每个值有且只有一个 owner（所有者）
2. owner 离开作用域，值自动销毁
3. 值可以 move（转移）或 borrow（借用）

### Move（转移）

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 的所有权 move 给 s2

println!("{}", s1);  // ❌ 编译错误！s1 没了
```

函数调用也会 move：

```rust
fn take_ownership(s: String) {
    println!("{}", s);
}

let my_string = String::from("hello");
take_ownership(my_string);  // 所有权 move 进函数
println!("{}", my_string);  // ❌ 编译错误！
```

### Borrow（借用）

不想 move？用引用：

```rust
fn print_string(s: &String) {  // & 表示借用
    println!("{}", s);
}

let my_string = String::from("hello");
print_string(&my_string);  // 借给函数用
println!("{}", my_string);  // ✅ 还能用
```

可变借用：

```rust
fn add_world(s: &mut String) {
    s.push_str(", world!");
}

let mut my_string = String::from("hello");
add_world(&mut my_string);
```

### 借用规则

同一时间只能有：
- 多个不可变引用 `&T`（都只读，OK）
- 或一个可变引用 `&mut T`（只有一个人写）
- **不能同时有！**

---

## 第4课：生命周期 Lifetime

### 解决什么问题？

防止悬垂引用（dangling reference）：

```rust
let r;
{
    let x = 5;
    r = &x;
}  // x 死了
println!("{}", r);  // ❌ 编译错误！
```

### 大部分时候不用写

编译器自动推断（生命周期省略规则）。

### 什么时候必须手写？

编译器搞不清返回的引用跟哪个输入有关：

```rust
fn longer<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() { s1 } else { s2 }
}
```

`'a` 的意思：返回值不会比 s1 和 s2 中较短命的那个活得更久。

### 结构体里存引用

```rust
struct Request<'a> {
    url: &'a str,
}
```

### 'static

活得和程序一样久：

```rust
let s: &'static str = "hello";  // 字符串字面量
```

---

## 第5课：错误处理

### Rust 没有 try-catch

错误就是普通的返回值。

### Option：可能有值，可能没有

```rust
enum Option<T> {
    Some(T),
    None,
}

let numbers = vec![1, 2, 3];
match numbers.get(10) {
    Some(n) => println!("找到: {}", n),
    None => println!("没找到"),
}
```

### Result：可能成功，可能失败

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}

match fs::read_to_string("config.json") {
    Ok(content) => println!("{}", content),
    Err(e) => println!("读取失败: {}", e),
}
```

### `?` 操作符

错误传播神器：

```rust
fn get_user_name(id: u32) -> Result<String, DbError> {
    let conn = connect_db()?;       // 失败直接返回 Err
    let user = conn.find_user(id)?; // 失败直接返回 Err
    Ok(user.name)
}
```

### 常用方法

| 方法 | 用途 |
|------|------|
| `unwrap()` | 直接取值，None/Err 时 panic |
| `expect("msg")` | unwrap + 自定义错误信息 |
| `unwrap_or(default)` | 给默认值 |
| `map()` | 链式转换 |

---

## 第6课：Rust 设计模式

### 核心区别：没有传统 OOP

Rust **没有继承**，用组合 + Trait 代替。

### 模式1：用 Enum 代替继承层次

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => 3.14 * radius * radius,
            Shape::Rectangle { width, height } => width * height,
        }
    }
}
```

### 模式2：Trait 代替接口

```rust
trait Drawable {
    fn draw(&self);
}

impl Drawable for Circle {
    fn draw(&self) { ... }
}
```

**可以给别人的类型实现 Trait！** Java/PHP 做不到。

### 模式3：Builder 模式

Rust 没有默认参数、没有重载，构造复杂对象靠 Builder：

```rust
let server = ServerBuilder::new()
    .host("0.0.0.0")
    .port(3000)
    .build();
```

### 模式4：Newtype —— 防止搞混

```rust
struct UserName(String);
struct Email(String);

fn create_user(name: UserName, email: Email) { ... }
// 写反会编译错误
```

### 模式5：类型状态（Type State）

```rust
impl Order<Draft> {
    fn submit(self) -> Order<Submitted> { ... }
}

impl Order<Submitted> {
    fn pay(self) -> Order<Paid> { ... }
}

// Draft 状态调用 pay() → 编译错误
```

---

## 第7课：Rust 特殊语法

### `*` 解引用

```rust
let x = 5;
let r = &x;
println!("{}", *r);  // 解引用，取出值 5

let mut y = 10;
let r2 = &mut y;
*r2 = 20;  // 通过引用修改
```

### `&` 借用（不是位运算）

```rust
let x = 5;
let r = &x;      // 不可变借用
let r2 = &mut x; // 可变借用
```

### `::` 路径分隔符

```rust
String::from("hello")
std::io::stdin()
Option::Some(5)
```

### `->` 返回类型

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

### `|x|` 闭包

```rust
let double = |x| x * 2;
vec![1,2,3].iter().map(|x| x * 2);
```

### `;` 分号影响返回值

```rust
fn five() -> i32 {
    5      // ✅ 返回 5（表达式）
}

fn five_wrong() -> i32 {
    5;     // ❌ 编译错误！加了分号变成语句
}
```

### `'a` 生命周期

```rust
fn longest<'a>(s1: &'a str, s2: &'a str) -> &'a str
```

### `?` 错误传播

```rust
let content = fs::read_to_string("file.txt")?;
```

### `_` 忽略值

```rust
let _unused = 42;
let (x, _, z) = (1, 2, 3);
```

### `..` 范围

```rust
for i in 0..5 { }   // 0,1,2,3,4
for i in 0..=5 { }  // 0,1,2,3,4,5
```

---

## 第8课：Enum 实战场景

### API 响应状态

```rust
enum ApiResponse<T> {
    Success(T),
    Error { code: u32, message: String },
    Loading,
}
```

### 订单状态机

```rust
enum OrderStatus {
    Pending,
    Paid { paid_at: DateTime, amount: f64 },
    Shipped { tracking_number: String },
    Delivered { delivered_at: DateTime },
    Cancelled { reason: String },
}
```

### 支付方式

```rust
enum PaymentMethod {
    CreditCard { last_four: String, brand: String },
    PayPal { email: String },
    Crypto { wallet_address: String, chain: String },
}
```

### 核心价值

| 问题 | 用字符串/数字 | 用 Enum |
|-----|--------------|---------|
| Typo | 运行时才发现 | 编译期报错 |
| 漏处理情况 | 可能忘记 | match 强制穷尽 |
| 每种状态带不同数据 | 塞一堆 nullable 字段 | 每个变体精确定义 |

---

## 第9课：泛型 Generics

### 泛型函数

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

### 泛型结构体

```rust
struct Point<T> {
    x: T,
    y: T,
}

struct Point2<T, U> {
    x: T,
    y: U,
}
```

### `<>` 常见问题

1. **嵌套太深** → 用 `type` 别名
2. **Trait bound 太长** → 用 `where` 子句
3. **dyn 大小不确定** → 用 `Box`/`Arc` 包装
4. **类型推断失败** → 显式标注或 turbofish `::<>`

---

## 第10课：Trait 入门

### 定义 Trait

```rust
trait Greet {
    fn say_hello(&self) -> String;
}
```

### 实现 Trait

```rust
impl Greet for Person {
    fn say_hello(&self) -> String {
        format!("你好，我是 {}！", self.name)
    }
}
```

### Trait 约束

```rust
fn greet_twice<T: Greet>(item: &T) {
    println!("{}", item.say_hello());
}
```

### 常见标准库 Trait

| Trait | 作用 |
|-------|------|
| `Debug` | 调试打印 `{:?}` |
| `Clone` | 显式深拷贝 |
| `Copy` | 隐式复制 |
| `PartialEq` | `==` 比较 |
| `Display` | 用户友好打印 |

### derive 自动实现

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point { x: i32, y: i32 }
```

---

## 第11课：Trait 对象与 dyn

### 什么时候用？

不同类型放同一个集合：

```rust
let shapes: Vec<Box<dyn Draw>> = vec![
    Box::new(Circle { radius: 5.0 }),
    Box::new(Rectangle { width: 10.0, height: 3.0 }),
];
```

### 静态 vs 动态分发

| 对比 | 静态 `impl Trait` | 动态 `dyn Trait` |
|------|------------------|-----------------|
| 决定时机 | 编译时 | 运行时 |
| 性能 | 更快 | 有虚函数表开销 |
| 灵活性 | 单一类型 | 多种类型混合 |

### 对象安全

方法不能返回 `Self`，不能有泛型参数。

---

## 第12课：Vec 动态数组

```rust
let mut v = vec![1, 2, 3];

v.push(4);           // 添加
let first = v[0];    // 索引访问
let second = v.get(1); // 返回 Option，更安全

for item in &v { }   // 遍历
for item in &mut v { *item += 1; }  // 可变遍历
```

---

## 第13课：String 深入

### 两种字符串

| 类型 | 存储位置 | 可变性 |
|------|---------|--------|
| `&str` | 栈/静态区 | 不可变 |
| `String` | 堆 | 可变 |

### 拼接

```rust
s.push_str("world");     // 修改原 String
let s3 = s1 + &s2;       // + 消耗 s1
let s3 = format!("{} {}", s1, s2);  // 最灵活
```

### 不能用索引！

```rust
let s = String::from("你好");
// s[0]  // ❌ 编译错误！UTF-8 编码

for c in s.chars() { }  // 按字符遍历
```

---

## 第14课：HashMap

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert("蓝队", 10);

// 获取
let score = scores.get("蓝队");  // Option<&V>

// 不存在才插入
scores.entry("绿队").or_insert(30);

// 词频统计
let count = word_count.entry(word).or_insert(0);
*count += 1;
```

---

## 第15课：迭代器 Iterators

### 适配器（懒求值）

```rust
let result: Vec<i32> = v.iter()
    .filter(|x| *x > 2)
    .map(|x| x * 10)
    .collect();
```

### 消费者

```rust
v.iter().sum()      // 求和
v.iter().count()    // 计数
v.iter().find(|x| *x > 3)  // 查找
v.iter().any(|x| x % 2 == 0)  // 存在
```

---

## 第16课：闭包 Closures

### 基本语法

```rust
let add = |a, b| a + b;
let double = |x| x * 2;
```

### 捕获环境

```rust
let factor = 10;
let multiply = |x| x * factor;  // 捕获外部变量
```

### 三个 Trait

| Trait | 捕获方式 | 调用次数 |
|-------|---------|---------|
| `Fn` | 不可变借用 | 多次 |
| `FnMut` | 可变借用 | 多次 |
| `FnOnce` | 移动（消耗）| 一次 |

### move 关键字

```rust
thread::spawn(move || {
    println!("{:?}", data);  // data 移入闭包
});
```

---

## 第17课：Box 堆分配

```rust
let b = Box::new(5);  // 堆上分配

// 递归类型
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

---

## 第18课：Rc 引用计数

```rust
use std::rc::Rc;

let data = Rc::new(String::from("hello"));
let a = Rc::clone(&data);  // 引用计数 +1
let b = Rc::clone(&data);

println!("引用计数: {}", Rc::strong_count(&data));
```

**限制**：只能单线程，只读。

---

## 第19课：RefCell 内部可变性

```rust
use std::cell::RefCell;

let data = RefCell::new(5);
*data.borrow_mut() += 10;  // 运行时借用检查
println!("{}", data.borrow());
```

### Rc + RefCell = 多所有者 + 可变

```rust
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
shared.borrow_mut().push(4);
```

---

## 第20课：Weak 与循环引用

### 问题

Rc 互相引用会内存泄漏。

### 解决

用 `Weak<T>` 弱引用：

```rust
use std::rc::{Rc, Weak};

let weak: Weak<i32> = Rc::downgrade(&strong);

if let Some(rc) = weak.upgrade() {
    println!("值还在: {}", rc);
}
```

**设计原则**：「拥有」用强引用，「访问」用弱引用。

---

## 第21课：线程基础

```rust
use std::thread;

let handle = thread::spawn(move || {
    println!("子线程");
});

handle.join().unwrap();  // 等待完成
```

**move**：强制闭包获取变量所有权。

---

## 第22课：Channel 消息传递

```rust
use std::sync::mpsc;

let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    tx.send("hello").unwrap();
});

let received = rx.recv().unwrap();
```

- `tx.clone()` 创建多个发送端
- `rx` 可当迭代器用

---

## 第23课：Mutex 互斥锁

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));

let counter_clone = Arc::clone(&counter);
thread::spawn(move || {
    let mut num = counter_clone.lock().unwrap();
    *num += 1;
});
```

**Arc + Mutex** = 多线程共享可变数据。

---

## 第24课：RwLock 读写锁

```rust
use std::sync::RwLock;

let lock = RwLock::new(5);

let r1 = lock.read().unwrap();   // 多个读者
let mut w = lock.write().unwrap();  // 独占写入
```

**适合**：读多写少的场景。

---

## 第25课：Send 与 Sync

| Trait | 含义 |
|-------|------|
| Send | 所有权可跨线程转移 |
| Sync | 引用可被多线程共享 |

- `Rc` 不是 Send（非原子计数）
- `Arc` 是 Send + Sync

---

## 第26课：原子类型 Atomics

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

let counter = AtomicUsize::new(0);
counter.fetch_add(1, Ordering::SeqCst);
```

**适合**：计数器、标志位、无锁数据结构。

---

## 第27课：async/await 入门

```rust
async fn fetch_data() -> String {
    "data".to_string()
}

#[tokio::main]
async fn main() {
    let data = fetch_data().await;
}
```

- `async fn` 返回 `Future`
- `.await` 才执行
- 需要 runtime（tokio）

---

## Q&A 问答整理

### Q&A: Borrow 的好处

1. **不用复制，省内存**
2. **保持所有权，后面还能用**
3. **编译器帮你查并发 bug**

### Q&A: 多线程数据竞争怎么识别

不是"识别"，是**根本不让你写出来**。

核心机制：`Send` 和 `Sync` Trait，编译时检查。

### Q&A: 生命周期防住什么 Bug

| Bug 类型 | 症状 |
|---------|------|
| 悬垂指针 | 随机值/崩溃 |
| Use After Free | 访问已释放内存 |
| 迭代器失效 | 遍历时修改集合 |

### Q&A: Rust vs Laravel 吞吐量

| 框架 | 请求/秒 |
|-----|---------|
| Actix-web (Rust) | 400,000+ |
| Laravel | 1,000-3,000 |
| Laravel Octane | 5,000-15,000 |

**Rust 比 Laravel 快 100-300 倍**。

---

## 学习路线

- ✅ 基础语法
- ✅ 所有权 / 借用 / 生命周期
- ✅ 错误处理
- ✅ 泛型与 Trait
- ✅ 集合与迭代器
- ✅ 智能指针
- ✅ 并发编程
- ✅ 异步编程入门
- ⏳ async/await 进阶
- ⏳ 宏（Macros）
- ⏳ unsafe Rust
- ⏳ Web 框架实战（Axum/Actix）

---

*笔记整理：性奴001*
*最后更新：2026-02-16*
