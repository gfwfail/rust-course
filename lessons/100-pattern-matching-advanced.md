# 第 100 课：模式匹配进阶 — Rust 最强大的武器

🎉 **里程碑！** 恭喜大家坚持到第 100 课！

今天深入讲解 Rust 最核心的特性——**模式匹配**（Pattern Matching）。之前我们用过 `match` 和 `if let`，今天系统梳理所有模式语法，掌握高级技巧。

---

## 🎯 模式匹配的本质

模式匹配不只是 switch-case 的升级版，它是一种**解构**数据的方式。

```rust
// PHP/JS 思维：检查值
if ($x == 1) { ... }
else if ($x == 2) { ... }

// Rust 思维：解构 + 绑定
match x {
    1 => ...,
    n @ 2..=10 => println!("2-10 范围，值是 {}", n),
    _ => ...,
}
```

---

## 📦 模式可以出现在哪里？

```rust
// 1. match 表达式
match value {
    pattern => expression,
}

// 2. if let
if let pattern = value {
    // ...
}

// 3. while let
while let Some(x) = iter.next() {
    // ...
}

// 4. for 循环
for (index, value) in vec.iter().enumerate() {
    // (index, value) 就是模式！
}

// 5. let 语句
let (x, y) = (1, 2);  // 解构元组
let Point { x, y } = point;  // 解构结构体

// 6. 函数参数
fn print_point(&(x, y): &(i32, i32)) {
    println!("x={}, y={}", x, y);
}
```

---

## 🔧 模式语法全览

### 1. 字面量模式

```rust
let x = 1;
match x {
    1 => println!("一"),
    2 => println!("二"),
    _ => println!("其他"),
}
```

### 2. 变量模式（绑定）

```rust
let x = 5;
match x {
    n => println!("值是 {}", n),  // n 绑定了 x 的值
}

// ⚠️ 注意：变量模式会"遮蔽"外部变量
let x = Some(5);
let y = 10;

match x {
    Some(50) => println!("50"),
    Some(y) => println!("y = {}", y),  // 这里 y 是新绑定，不是外部的 10！
    _ => println!("其他"),
}
// 输出: y = 5
```

### 3. 通配符模式

```rust
// _ 忽略整个值
match value {
    _ => println!("不管是啥"),
}

// _x 忽略但消除警告
fn foo(_x: i32) {}  // 不使用 _x 不会警告

// .. 忽略剩余部分
let (first, .., last) = (1, 2, 3, 4, 5);
// first = 1, last = 5

struct Point { x: i32, y: i32, z: i32 }
let Point { x, .. } = point;  // 只关心 x
```

### 4. 范围模式

```rust
let x = 5;
match x {
    1..=5 => println!("1 到 5"),  // 包含边界
    6..=10 => println!("6 到 10"),
    _ => println!("其他"),
}

// 字符范围
let c = 'c';
match c {
    'a'..='z' => println!("小写字母"),
    'A'..='Z' => println!("大写字母"),
    _ => println!("其他"),
}
```

### 5. 解构模式

```rust
// 解构元组
let (x, y, z) = (1, 2, 3);

// 解构结构体
struct User { name: String, age: u32 }
let user = User { name: "Alice".into(), age: 30 };

let User { name, age } = user;
// 或者重命名
let User { name: n, age: a } = user;

// 解构枚举
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn process(msg: Message) {
    match msg {
        Message::Quit => println!("退出"),
        Message::Move { x, y } => println!("移动到 ({}, {})", x, y),
        Message::Write(text) => println!("写入: {}", text),
        Message::ChangeColor(r, g, b) => println!("颜色: RGB({},{},{})", r, g, b),
    }
}

// 嵌套解构
let ((feet, inches), Point { x, y }) = ((3, 10), Point { x: 5, y: -3 });
```

### 6. | 模式（或）

```rust
let x = 1;
match x {
    1 | 2 | 3 => println!("1, 2, 或 3"),
    4 | 5 => println!("4 或 5"),
    _ => println!("其他"),
}

// 也可用于枚举
enum Color { Red, Green, Blue, Yellow }
match color {
    Color::Red | Color::Yellow => println!("暖色"),
    Color::Green | Color::Blue => println!("冷色"),
}
```

### 7. @ 绑定

```rust
// 匹配范围的同时绑定变量
let x = 5;
match x {
    n @ 1..=5 => println!("1-5 范围内，值是 {}", n),
    n @ 6..=10 => println!("6-10 范围内，值是 {}", n),
    _ => println!("范围外"),
}

// 解构的同时保留整体
enum Message {
    Hello { id: i32 },
}

let msg = Message::Hello { id: 5 };

match msg {
    Message::Hello { id: id_var @ 3..=7 } => {
        println!("id 在 3-7 范围: {}", id_var);
    }
    Message::Hello { id } => {
        println!("其他 id: {}", id);
    }
}
```

---

## 🛡️ Match Guard（匹配守卫）

当模式不够用时，加个条件：

```rust
let num = Some(4);

match num {
    Some(x) if x < 5 => println!("小于 5: {}", x),
    Some(x) if x >= 5 => println!("大于等于 5: {}", x),
    None => println!("None"),
    _ => unreachable!(),  // 理论上不会到这
}

// 结合 | 使用（守卫应用于所有分支）
let x = 4;
let y = false;

match x {
    4 | 5 | 6 if y => println!("yes"),  // 4|5|6 且 y 为 true
    _ => println!("no"),
}
// 输出: no（因为 y 是 false）
```

---

## 🎭 ref 和 ref mut

在模式中借用而非移动：

```rust
let s = String::from("hello");

// ❌ 这会移动 s
// let Some(inner) = Some(s);

// ✅ 使用 ref 借用
match Some(s) {
    Some(ref inner) => println!("{}", inner),
    None => {},
}
// s 仍然有效（没有被移动）

// ref mut 可变借用
let mut s = Some(String::from("hello"));

match s {
    Some(ref mut inner) => {
        inner.push_str(" world");
    }
    None => {},
}
println!("{:?}", s);  // Some("hello world")
```

**注意**：现代 Rust（2018+）在 `match` 中自动处理借用，大多数情况不需要显式 `ref`。

---

## 💡 实战技巧

### 技巧 1：用 matches! 宏简化布尔判断

```rust
// 冗长写法
let is_letter = match c {
    'a'..='z' | 'A'..='Z' => true,
    _ => false,
};

// ✅ 简洁写法
let is_letter = matches!(c, 'a'..='z' | 'A'..='Z');

// 带 guard
let is_even_positive = matches!(x, n if n > 0 && n % 2 == 0);
```

### 技巧 2：用 let-else 处理失败情况

```rust
// Rust 1.65+ 的 let-else 语法
fn get_count(input: &str) -> Option<usize> {
    let Ok(count) = input.parse::<usize>() else {
        return None;  // 解析失败直接返回
    };
    Some(count * 2)
}

// 等价于
fn get_count_old(input: &str) -> Option<usize> {
    let count = match input.parse::<usize>() {
        Ok(n) => n,
        Err(_) => return None,
    };
    Some(count * 2)
}
```

### 技巧 3：解构复杂嵌套结构

```rust
struct Wrapper {
    data: Option<Result<(i32, String), String>>,
}

let w = Wrapper {
    data: Some(Ok((42, "hello".into()))),
};

// 一步到位解构
if let Wrapper { 
    data: Some(Ok((num, ref text))) 
} = w {
    println!("num={}, text={}", num, text);
}
```

### 技巧 4：穷尽性检查

```rust
enum State {
    Active,
    Inactive,
    Pending,
}

// 编译器确保覆盖所有情况
fn describe(s: State) -> &'static str {
    match s {
        State::Active => "活跃",
        State::Inactive => "不活跃",
        // 如果忘了 Pending，编译器会报错！
        State::Pending => "待定",
    }
}

// 用 #[non_exhaustive] 标记的枚举强制使用 _
```

---

## 🔄 对比其他语言

```rust
// Rust 模式匹配
match result {
    Ok(value) => process(value),
    Err(Error::NotFound) => handle_not_found(),
    Err(Error::Permission(msg)) => eprintln!("权限: {}", msg),
    Err(e) => return Err(e),
}

// PHP 需要手动解构
// switch ($result['type']) {
//     case 'ok': process($result['value']); break;
//     case 'error':
//         switch ($result['error']) { ... }
// }

// JS 稍微好点但还是繁琐
// if (result.ok) { process(result.value); }
// else if (result.error === 'not_found') { ... }
```

**Rust 优势**：
1. 编译器确保穷尽（不会漏处理情况）
2. 解构和绑定一步到位
3. 类型安全，没有运行时错误

---

## 🧠 课后练习

1. 用模式匹配实现一个简单的命令解析器：

```rust
enum Command {
    Get { key: String },
    Set { key: String, value: String },
    Delete { key: String },
    List,
}

fn parse(input: &str) -> Option<Command> {
    // 实现解析逻辑
    todo!()
}
```

2. 使用 `matches!` 宏判断一个字符是否是十六进制字符（0-9, a-f, A-F）

3. 用 `let-else` 重写这段代码：
```rust
fn double_positive(s: &str) -> Option<i32> {
    let n: i32 = s.parse().ok()?;
    if n > 0 {
        Some(n * 2)
    } else {
        None
    }
}
```

---

*🎉 第 100 课完！恭喜大家走过了 100 节课的 Rust 学习之旅！*
