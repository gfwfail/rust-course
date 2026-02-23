# 第 89 课：std::any — 运行时类型信息 (RTTI)

> 日期：2026-02-23  
> 主题：Any trait、TypeId、类型擦除后的复原

---

## 📌 问题：Rust 怎么没有反射？

如果你从 PHP/Java 过来，可能习惯了这样写：

```php
// PHP
$obj = new SomeClass();
get_class($obj);           // "SomeClass"
$obj instanceof SomeClass; // true
```

Rust 是静态类型语言，编译后**类型信息大部分都没了**。但有时候我们确实需要在运行时判断类型——这就是 `std::any` 的用武之地。

---

## 🔧 std::any 模块概览

```rust
use std::any::{
    Any,        // 核心 trait
    TypeId,     // 类型的唯一标识符
    type_name,  // 获取类型名（调试用）
};
```

---

## 📦 Any trait — 类型擦除的基础

### 定义

```rust
pub trait Any: 'static {
    fn type_id(&self) -> TypeId;
}
```

**注意 `'static` 约束**：只有不含非静态引用的类型才能实现 `Any`。

### 自动实现

`Any` 对所有 `T: 'static` 类型**自动实现**：

```rust
// 这些都自动实现了 Any
i32          // ✅
String       // ✅
Vec<u8>      // ✅
MyStruct     // ✅ (只要没有非 'static 引用)

// 这些不行
&'a str      // ❌ 有生命周期参数
&'a MyStruct // ❌ 同上
```

---

## 🆔 TypeId — 类型的指纹

每个类型在编译期都会生成一个唯一的 `TypeId`：

```rust
use std::any::TypeId;

let id_i32 = TypeId::of::<i32>();
let id_string = TypeId::of::<String>();

println!("{:?}", id_i32);    // TypeId { t: 某个哈希值 }
println!("{:?}", id_string); // TypeId { t: 另一个哈希值 }

// 比较类型
assert_ne!(id_i32, id_string);
assert_eq!(TypeId::of::<i32>(), TypeId::of::<i32>());
```

### 实际用途

```rust
use std::any::{Any, TypeId};

fn is_string(val: &dyn Any) -> bool {
    val.type_id() == TypeId::of::<String>()
}

fn main() {
    let s = String::from("hello");
    let n = 42i32;
    
    println!("{}", is_string(&s)); // true
    println!("{}", is_string(&n)); // false
}
```

---

## 🔄 downcast — 从 Any 还原具体类型

这是 `Any` 最重要的功能：**类型擦除后的复原**。

### downcast_ref / downcast_mut

```rust
use std::any::Any;

fn print_if_string(val: &dyn Any) {
    // 尝试转换为 &String
    if let Some(s) = val.downcast_ref::<String>() {
        println!("It's a string: {}", s);
    } else {
        println!("Not a string");
    }
}

fn main() {
    let s = String::from("hello");
    let n = 42i32;
    
    print_if_string(&s); // It's a string: hello
    print_if_string(&n); // Not a string
}
```

### Box 的 downcast

对于 `Box<dyn Any>`，可以直接 downcast 拿回所有权：

```rust
use std::any::Any;

fn main() {
    let boxed: Box<dyn Any> = Box::new(42i32);
    
    // 尝试 downcast，成功返回 Ok(Box<i32>)
    match boxed.downcast::<i32>() {
        Ok(i) => println!("Got i32: {}", i),
        Err(original) => println!("Not i32, got back the box"),
    }
}
```

### downcast 的完整 API

```rust
impl dyn Any {
    pub fn is<T: Any>(&self) -> bool;
    pub fn downcast_ref<T: Any>(&self) -> Option<&T>;
    pub fn downcast_mut<T: Any>(&mut self) -> Option<&mut T>;
}

impl Box<dyn Any> {
    pub fn downcast<T: Any>(self) -> Result<Box<T>, Box<dyn Any>>;
}
```

---

## 📛 type_name — 调试友好的类型名

```rust
use std::any::type_name;

fn print_type<T>(_: &T) {
    println!("{}", type_name::<T>());
}

fn main() {
    print_type(&42i32);        // "i32"
    print_type(&"hello");      // "&str"
    print_type(&vec![1, 2]);   // "alloc::vec::Vec<i32>"
    print_type(&Some(42));     // "core::option::Option<i32>"
}
```

⚠️ **注意**：`type_name` 返回的字符串**不保证稳定**！不同编译器版本可能返回不同格式。只用于调试，不要用于逻辑判断。

---

## 🎯 实战：类型安全的异构容器

场景：你想存储不同类型的值，但又想类型安全地取出来。

### 方法一：枚举（推荐）

```rust
enum Value {
    Int(i32),
    Float(f64),
    Text(String),
}

fn main() {
    let values: Vec<Value> = vec![
        Value::Int(42),
        Value::Float(3.14),
        Value::Text("hello".into()),
    ];
    
    for v in values {
        match v {
            Value::Int(i) => println!("int: {}", i),
            Value::Float(f) => println!("float: {}", f),
            Value::Text(s) => println!("text: {}", s),
        }
    }
}
```

### 方法二：Any（当类型不可枚举时）

```rust
use std::any::Any;
use std::collections::HashMap;

struct TypeMap {
    data: HashMap<std::any::TypeId, Box<dyn Any>>,
}

impl TypeMap {
    fn new() -> Self {
        Self { data: HashMap::new() }
    }
    
    fn insert<T: 'static>(&mut self, val: T) {
        self.data.insert(std::any::TypeId::of::<T>(), Box::new(val));
    }
    
    fn get<T: 'static>(&self) -> Option<&T> {
        self.data
            .get(&std::any::TypeId::of::<T>())
            .and_then(|b| b.downcast_ref())
    }
}

fn main() {
    let mut map = TypeMap::new();
    map.insert(42i32);
    map.insert(String::from("hello"));
    
    println!("{:?}", map.get::<i32>());    // Some(42)
    println!("{:?}", map.get::<String>()); // Some("hello")
    println!("{:?}", map.get::<f64>());    // None
}
```

这种模式在 Web 框架中很常见（存储请求扩展、中间件数据等）。

---

## 🔍 Any + Send + Sync

多线程场景需要加约束：

```rust
use std::any::Any;

// 单线程版
type AnyBox = Box<dyn Any>;

// 多线程版
type AnyBoxSend = Box<dyn Any + Send>;
type AnyBoxSendSync = Box<dyn Any + Send + Sync>;

fn store_thread_safe(val: Box<dyn Any + Send + Sync>) {
    // 可以跨线程传递和共享
}
```

---

## ⚡ 性能考虑

```rust
// TypeId 比较是 O(1)，就是两个 u64 比较
// downcast 内部也是 TypeId 比较 + 指针转换

// 但是！相比编译期泛型，运行时检查有开销：
// 1. 虚表调用（dyn Any）
// 2. 分支判断（downcast 可能失败）
// 3. 可能的 Box 分配

// 如果能用泛型解决，优先用泛型
fn generic<T: Debug>(val: T) { ... }  // 编译期单态化，零开销

// Any 适合真正需要"运行时多态"的场景
```

---

## 🎓 PHP/Laravel 对比

```php
// PHP 反射无处不在
$obj = new User();
get_class($obj);                    // "User"
$obj instanceof User;               // true
(new ReflectionClass($obj))->getMethods();

// Laravel 的服务容器大量使用反射
app()->make(UserService::class);    // 自动注入依赖
```

```rust
// Rust 的 Any 功能有限得多：
// ❌ 不能获取字段列表
// ❌ 不能调用方法
// ❌ 不能动态实例化
// ✅ 只能判断类型 + downcast

// 这是设计选择：Rust 追求零成本抽象
// 完整反射需要保留大量元数据，会增加二进制大小和运行时开销
```

---

## 📝 小测验

```rust
use std::any::Any;

fn main() {
    let vals: Vec<Box<dyn Any>> = vec![
        Box::new(42i32),
        Box::new("hello"),  // 这是什么类型？
        Box::new(String::from("world")),
    ];
    
    // Q1: vals[0].downcast_ref::<i32>() 返回？
    // A1: Some(&42)
    
    // Q2: vals[1].downcast_ref::<String>() 返回？
    // A2: None！"hello" 是 &str，不是 String
    
    // Q3: vals[1].downcast_ref::<&str>() 返回？
    // A3: Some(&"hello")
    
    // Q4: 如何判断 vals[2] 是不是 String？
    // A4: vals[2].is::<String>()  → true
}
```

---

## 🔑 总结

| API | 用途 |
|-----|------|
| `TypeId::of::<T>()` | 获取类型的唯一标识符 |
| `val.type_id()` | 获取 `dyn Any` 值的类型标识符 |
| `val.is::<T>()` | 检查是否为特定类型 |
| `val.downcast_ref::<T>()` | 尝试转换为 `&T` |
| `val.downcast_mut::<T>()` | 尝试转换为 `&mut T` |
| `box.downcast::<T>()` | 尝试转换 Box，恢复所有权 |
| `type_name::<T>()` | 获取类型名字符串（调试用） |

**核心理解**：
- `Any` 是 Rust 中有限的运行时类型信息
- 主要用于**类型擦除后的复原**（如异构容器）
- 相比 PHP/Java 的反射，功能非常有限
- 优先使用泛型和枚举，只在必要时用 `Any`

---

**下节预告**: `std::convert` — From, Into, TryFrom, TryInto 类型转换家族 🔄
