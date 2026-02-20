# 第 65 课：模式匹配深入 (Pattern Matching Deep Dive)

> 日期：2026-02-20  
> 主题：回归语言本身，深入理解 Rust 核心特性

---

## 🎯 模式匹配不只是 match

PHP/JS 里的 switch 只能匹配值：
```javascript
switch (x) {
    case 1: break;
    case 2: break;
}
```

Rust 的模式匹配可以**解构**、**绑定**、**守卫**：
```rust
match value {
    // 解构 struct
    Point { x: 0, y } => println!("在 y 轴上, y = {}", y),
    
    // 解构 tuple
    (first, _, third) => println!("first={}, third={}", first, third),
    
    // 范围匹配
    1..=5 => println!("1到5之间"),
    
    // 守卫条件
    n if n < 0 => println!("负数"),
    
    // 绑定 + 条件
    name @ "Alice" | name @ "Bob" => println!("VIP: {}", name),
}
```

---

## 📍 模式可以出现的 6 个地方

```rust
// 1️⃣ match 表达式
match x {
    1 => "one",
    _ => "other",
}

// 2️⃣ if let
if let Some(value) = option {
    println!("{}", value);
}

// 3️⃣ while let
while let Some(item) = stack.pop() {
    println!("{}", item);
}

// 4️⃣ for 循环
for (index, value) in vec.iter().enumerate() {
    println!("{}: {}", index, value);
}

// 5️⃣ let 语句（是的，let 就是模式匹配！）
let (x, y, z) = (1, 2, 3);
let Point { x, y } = point;

// 6️⃣ 函数参数
fn print_point(&(x, y): &(i32, i32)) {
    println!("({}, {})", x, y);
}
```

**重点**：`let x = 5;` 本质上就是模式匹配，`x` 是一个模式！

---

## 🔧 解构全家桶

### Struct 解构
```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}

let user = User {
    name: String::from("Alice"),
    age: 30,
    active: true,
};

// 完整解构
let User { name, age, active } = user;

// 部分解构，用 .. 忽略其余
let User { name, .. } = user;

// 重命名
let User { name: user_name, age: user_age, .. } = user;
```

### Enum 解构
```rust
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
        Message::ChangeColor(r, g, b) => println!("RGB({}, {}, {})", r, g, b),
    }
}
```

### 嵌套解构（一步到位）
```rust
struct Point { x: i32, y: i32 }

enum Shape {
    Circle { center: Point, radius: f64 },
    Rectangle { top_left: Point, bottom_right: Point },
}

let shape = Shape::Circle {
    center: Point { x: 0, y: 0 },
    radius: 10.0,
};

// 嵌套解构，直接拿到 x, y
match shape {
    Shape::Circle { center: Point { x, y }, radius } => {
        println!("圆心 ({}, {}), 半径 {}", x, y, radius);
    }
    Shape::Rectangle { top_left: Point { x: x1, y: y1 }, .. } => {
        println!("左上角 ({}, {})", x1, y1);
    }
}
```

---

## 🛡️ Match Guard（守卫条件）

当简单模式不够用时，加 `if` 条件：
```rust
let num = Some(4);

match num {
    Some(x) if x < 5 => println!("小于5: {}", x),
    Some(x) if x >= 5 => println!("大于等于5: {}", x),
    None => println!("无值"),
    _ => unreachable!(),
}

// 实际场景：权限检查
match (user.role, action) {
    (Role::Admin, _) => allow(),
    (Role::User, Action::Read) => allow(),
    (Role::User, Action::Write) if user.owns(&resource) => allow(),
    _ => deny(),
}
```

---

## 📎 @ 绑定（捕获匹配的值）

有时候你想匹配一个范围/条件，同时又要使用那个值：
```rust
enum Message {
    Hello { id: i32 },
}

let msg = Message::Hello { id: 5 };

match msg {
    // 匹配 id 在 3..=7 范围，同时绑定到 id_var
    Message::Hello { id: id_var @ 3..=7 } => {
        println!("id 在范围内: {}", id_var);
    }
    Message::Hello { id: 10..=12 } => {
        println!("id 在另一个范围");
        // 这里没法用 id，因为没绑定
    }
    Message::Hello { id } => {
        println!("其他 id: {}", id);
    }
}
```

**@** 的妙用：
```rust
// 匹配 Some，但只要非空字符串
match name {
    Some(s @ _) if !s.is_empty() => println!("名字: {}", s),
    _ => println!("无名"),
}

// 匹配数字范围并绑定
let age = 25;
match age {
    n @ 0..=17 => println!("未成年: {}", n),
    n @ 18..=65 => println!("成年: {}", n),
    n @ 66.. => println!("老年: {}", n),
}
```

---

## 💡 实战技巧

### 1. 用 matches! 宏简化判断
```rust
// 以前
let is_letter = match c {
    'a'..='z' | 'A'..='Z' => true,
    _ => false,
};

// 现在（标准库宏）
let is_letter = matches!(c, 'a'..='z' | 'A'..='Z');

// 判断 enum 变体
let is_some = matches!(option, Some(_));
let is_ok = matches!(result, Ok(_));
```

### 2. let-else（Rust 1.65+）
```rust
// 以前
let value = if let Some(v) = option {
    v
} else {
    return Err("没有值");
};

// 现在
let Some(value) = option else {
    return Err("没有值");
};
```

### 3. 解构赋值（Rust 1.59+）
```rust
let (mut x, mut y) = (1, 2);

// 交换！不需要临时变量
(x, y) = (y, x);

// 解构赋值到已有变量
struct Point { x: i32, y: i32 }
let mut point = Point { x: 0, y: 0 };
Point { x: point.x, y: point.y } = Point { x: 3, y: 4 };
```

---

## 🎓 课后练习

写一个函数，用模式匹配解析简单的命令：
```rust
enum Command {
    Quit,
    Move { x: i32, y: i32 },
    Say { message: String, loud: bool },
    Unknown(String),
}

fn describe(cmd: &Command) -> String {
    // 用一个 match 处理所有情况
    // 1. Quit -> "退出程序"
    // 2. Move { x: 0, y: 0 } -> "原地不动"
    // 3. Move { x, y } 其中 x == y -> "对角移动到 x"
    // 4. Move { x, y } 其他情况 -> "移动到 (x, y)"
    // 5. Say { loud: true, .. } -> "大声说: ..."
    // 6. Say { loud: false, .. } -> "轻声说: ..."
    // 7. Unknown(s) -> "未知命令: ..."
    todo!()
}
```

### 参考答案
```rust
fn describe(cmd: &Command) -> String {
    match cmd {
        Command::Quit => "退出程序".to_string(),
        Command::Move { x: 0, y: 0 } => "原地不动".to_string(),
        Command::Move { x, y } if x == y => format!("对角移动到 {}", x),
        Command::Move { x, y } => format!("移动到 ({}, {})", x, y),
        Command::Say { message, loud: true } => format!("大声说: {}", message),
        Command::Say { message, loud: false } => format!("轻声说: {}", message),
        Command::Unknown(s) => format!("未知命令: {}", s),
    }
}
```

---

## 📚 总结

模式匹配是 Rust 的灵魂特性之一：
- **解构**：直接从复杂类型中提取数据
- **守卫**：用 `if` 添加额外条件
- **@ 绑定**：匹配的同时绑定值
- **matches! 宏**：简化 bool 判断
- **let-else**：优雅处理失败情况

用好模式匹配，代码会优雅很多！

---

*下节预告：标准库深入 - std::iter 迭代器的隐藏方法*
