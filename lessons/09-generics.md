# 第9课：泛型 Generics

泛型让你写**一份代码，适用于多种类型**。不用为 `i32`、`f64`、`String` 各写一遍相同逻辑！

## 为什么需要泛型？

假设你要找出数组中最大的数：

```rust
fn largest_i32(list: &[i32]) -> &i32 {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn largest_f64(list: &[f64]) -> &f64 {
    // 完全一样的逻辑，只是类型不同...
}
```

重复代码 = 维护噩梦！

## 泛型函数

用 `<T>` 声明泛型参数，`T` 是占位符，代表"任意类型"：

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

fn main() {
    let nums = vec![1, 5, 3, 9, 2];
    println!("最大: {}", largest(&nums));  // 9
    
    let floats = vec![1.1, 5.5, 3.3];
    println!("最大: {}", largest(&floats)); // 5.5
}
```

> `T: PartialOrd` 是**约束**，表示 T 必须能比较大小。

## 泛型结构体

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let int_point = Point { x: 5, y: 10 };       // Point<i32>
    let float_point = Point { x: 1.5, y: 4.2 };  // Point<f64>
}
```

⚠️ **注意**：`x` 和 `y` 必须是**同一类型**，因为都是 `T`。

## 多个泛型参数

想让 x 和 y 类型不同？用多个泛型：

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

fn main() {
    let mixed = Point { x: 5, y: 3.14 };  // Point<i32, f64> ✅
}
```

## 给泛型结构体实现方法

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };
    println!("x = {}", p.x());
}
```

> 注意：`impl<T>` 声明泛型，然后 `Point<T>` 使用它。

## 只为特定类型实现方法

```rust
impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

这个方法**只有** `Point<f64>` 才能用！`Point<i32>` 调用会编译错误。

## 泛型枚举

你早就见过了：

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

## 性能：零成本抽象

Rust 在编译时会进行**单态化（Monomorphization）**：

```rust
let a = Some(5);    // 编译器生成 Option_i32
let b = Some(3.14); // 编译器生成 Option_f64
```

泛型代码会被展开成具体类型的代码，**运行时零开销**！

## `<>` 常见问题

### 1. 嵌套太深，编译器报错看不懂

```rust
HashMap<String, Vec<Arc<Mutex<Option<Result<User, Error>>>>>>
```

**解法**：用 `type` 别名简化

```rust
type SignerList = Vec<Arc<dyn ChainTransactionSigner>>;
type UserCache = HashMap<String, Arc<Mutex<User>>>;
```

### 2. Trait bound 地狱

```rust
// ❌ 越写越长
fn process<T: Clone + Send + Sync + Debug + Serialize + 'static>(data: T) { }

// ✅ 用 where 子句
fn process<T>(data: T) 
where
    T: Clone + Send + Sync + Debug + Serialize + 'static,
{
    // ...
}
```

### 3. 类型推断失败

```rust
// ❌ 编译器不知道你要什么类型
let numbers = vec![];
numbers.push(1);

// ✅ 明确告诉它
let numbers: Vec<i32> = vec![];
// 或
let numbers = Vec::<i32>::new();
```

### 4. Turbofish `::<>` 语法

有时候需要显式指定泛型：

```rust
let num = "42".parse::<i32>()?;
let set = HashSet::<String>::new();
```

长得像 🐟，所以叫 turbofish。

---

**下节课**：Trait 入门
