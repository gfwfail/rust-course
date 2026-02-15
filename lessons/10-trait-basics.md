# 第10课：Trait 入门

Trait 类似于其他语言的「接口」——它定义了一组方法签名，让不同类型可以共享同一套行为。

## 定义 Trait

```rust
trait Greet {
    fn say_hello(&self) -> String;
}
```

这里 `Greet` 定义了一个行为：任何实现它的类型都必须能 `say_hello`。

## 为类型实现 Trait

用 `impl Trait for Type` 语法：

```rust
struct Person {
    name: String,
}

struct Dog {
    name: String,
}

impl Greet for Person {
    fn say_hello(&self) -> String {
        format!("你好，我是 {}！", self.name)
    }
}

impl Greet for Dog {
    fn say_hello(&self) -> String {
        format!("汪汪！我叫 {}！", self.name)
    }
}

fn main() {
    let alice = Person { name: "Alice".to_string() };
    let buddy = Dog { name: "Buddy".to_string() };
    
    println!("{}", alice.say_hello()); // 你好，我是 Alice！
    println!("{}", buddy.say_hello()); // 汪汪！我叫 Buddy！
}
```

## Trait 作为参数（Trait Bound）

**限制泛型类型必须实现某个 Trait**：

```rust
// 语法糖写法（impl Trait）
fn greet_twice(item: &impl Greet) {
    println!("{}", item.say_hello());
    println!("{}", item.say_hello());
}

// 等价的完整泛型写法
fn greet_twice<T: Greet>(item: &T) {
    println!("{}", item.say_hello());
    println!("{}", item.say_hello());
}
```

现在 `greet_twice` 只接受实现了 `Greet` 的类型！

## 多个 Trait 约束

用 `+` 号组合多个约束：

```rust
use std::fmt::Display;

fn show_and_greet<T: Greet + Display>(item: &T) {
    println!("Display: {}", item);
    println!("Greet: {}", item.say_hello());
}
```

## where 子句

约束太多时，用 `where` 让签名更易读：

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Greet + Debug,
{
    // ...
}
```

## 默认实现

Trait 可以提供默认方法实现：

```rust
trait Greet {
    fn say_hello(&self) -> String {
        String::from("Hello!")  // 默认实现
    }
    
    fn say_goodbye(&self) -> String;  // 必须实现
}

impl Greet for Person {
    // say_hello 用默认的
    fn say_goodbye(&self) -> String {
        format!("再见，{}！", self.name)
    }
}
```

## 常见标准库 Trait

| Trait | 作用 |
|-------|------|
| `Debug` | 调试打印 `{:?}` |
| `Clone` | 显式深拷贝 `.clone()` |
| `Copy` | 隐式复制（栈上类型） |
| `PartialEq` | `==` 和 `!=` 比较 |
| `Eq` | 完全相等（需先实现 PartialEq） |
| `PartialOrd` | `<` `>` `<=` `>=` 比较 |
| `Ord` | 完全排序 |
| `Display` | 用户友好打印 `{}` |
| `Default` | 默认值 `Default::default()` |

## derive 宏自动实现

很多标准 Trait 可以自动派生：

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();
    
    println!("{:?}", p1);       // Point { x: 1, y: 2 }
    println!("{}", p1 == p2);   // true
}
```

## 返回实现了 Trait 的类型

用 `-> impl Trait` 返回：

```rust
fn make_greeter() -> impl Greet {
    Person { name: "匿名".to_string() }
}
```

注意：只能返回**同一种具体类型**，不能根据条件返回不同类型。

---

**下节课**：Trait 对象与 dyn
