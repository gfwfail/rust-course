# 第6课：Rust 设计模式

## 核心区别：没有传统 OOP

Rust **没有继承**。你不会写：
```java
// Java 思维
class Animal { }
class Dog extends Animal { }
```

Rust 用 **组合 + Trait** 代替继承。

## 模式1：用 Enum 代替继承层次

**Java/PHP 思维：**
```java
abstract class Shape { abstract double area(); }
class Circle extends Shape { ... }
class Rectangle extends Shape { ... }
```

**Rust 思维：**
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

**好处：**
- 编译器强制你处理所有情况（match 必须穷尽）
- 新增变体？所有 match 都会报错提醒你改
- 数据和行为分离，逻辑更清晰

## 模式2：Trait 代替接口/抽象类

**Java：**
```java
interface Drawable {
    void draw();
}
class Circle implements Drawable { ... }
```

**Rust：**
```rust
trait Drawable {
    fn draw(&self);
}

struct Circle { radius: f64 }

impl Drawable for Circle {
    fn draw(&self) {
        println!("画个圆，半径 {}", self.radius);
    }
}
```

**关键区别：可以给别人的类型实现 Trait！**

```rust
// 给标准库的 i32 加方法
trait Describable {
    fn describe(&self) -> String;
}

impl Describable for i32 {
    fn describe(&self) -> String {
        format!("这是数字 {}", self)
    }
}

// 现在 i32 有这个方法了
let n = 42;
println!("{}", n.describe());  // "这是数字 42"
```

Java/PHP 做不到给 String 类加方法吧？

## 模式3：Builder 模式（超级常用）

因为 Rust 没有默认参数、没有重载，构造复杂对象靠 Builder：

```rust
struct Server {
    host: String,
    port: u16,
    max_connections: usize,
    timeout_ms: u64,
}

struct ServerBuilder {
    host: String,
    port: u16,
    max_connections: usize,
    timeout_ms: u64,
}

impl ServerBuilder {
    fn new() -> Self {
        ServerBuilder {
            host: "localhost".into(),
            port: 8080,
            max_connections: 100,
            timeout_ms: 5000,
        }
    }
    
    fn host(mut self, h: &str) -> Self {
        self.host = h.into();
        self
    }
    
    fn port(mut self, p: u16) -> Self {
        self.port = p;
        self
    }
    
    fn build(self) -> Server {
        Server {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
            timeout_ms: self.timeout_ms,
        }
    }
}

// 使用
let server = ServerBuilder::new()
    .host("0.0.0.0")
    .port(3000)
    .build();
```

## 模式4：Newtype —— 包一层防止搞混

**问题场景：**
```rust
fn create_user(name: String, email: String) { ... }

// 容易写反！
create_user(email, name);  // 编译通过，但逻辑错了
```

**Newtype 解法：**
```rust
struct UserName(String);
struct Email(String);

fn create_user(name: UserName, email: Email) { ... }

// 现在写反会编译错误
create_user(Email("...".into()), UserName("...".into()));  // ❌ 类型不对
```

**一行 struct 就能防住一类 bug。**

## 模式5：类型状态（Type State）

用类型系统表示状态，错误使用直接编译不过：

```rust
struct Order<State> { 
    id: u64,
    _state: std::marker::PhantomData<State>,
}

struct Draft;
struct Submitted;
struct Paid;

impl Order<Draft> {
    fn submit(self) -> Order<Submitted> { ... }
}

impl Order<Submitted> {
    fn pay(self) -> Order<Paid> { ... }
}

impl Order<Paid> {
    fn ship(self) { ... }
}

// 使用
let order = Order::new();    // Draft
let order = order.submit();  // Submitted  
let order = order.pay();     // Paid
order.ship();                // ✅

// 错误使用
let order = Order::new();
order.pay();  // ❌ 编译错误！Draft 没有 pay 方法
```

**状态机在类型里，不可能走错流程。**

## 模式6：零成本抽象——泛型而不是 interface{}

**Go 的写法：**
```go
func process(data interface{}) {
    // 运行时类型检查
}
```

**Rust 的写法：**
```rust
fn process<T: Display>(data: T) {
    // 编译时确定具体类型，零运行时开销
    println!("{}", data);
}
```

泛型在编译时"单态化"，生成具体类型的代码，没有虚函数表、没有运行时分发。

## 思维转变总结

| OOP 思维 | Rust 思维 |
|---------|----------|
| 继承 | 组合 + Trait |
| 类层次 | Enum + match |
| 接口 | Trait（还能给别人的类型加） |
| 可选参数 | Builder 模式 |
| 运行时类型检查 | 编译时泛型 |
| 防御性编程 | 让错误代码编译不过 |

**Rust 核心哲学**：
> 能在编译期检查的，就不要留到运行时。

---

**下节课**：Rust 特殊语法
