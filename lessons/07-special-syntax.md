# 第7课：Rust 特殊语法

Rust 有很多和其他语言不同的语法符号，这里集中讲解。

## `*` 解引用 —— 不是指针乘法

**JS/PHP 里 `*` 就是乘法，但 Rust 里有两个意思：**

### 乘法（和其他语言一样）
```rust
let a = 5 * 3;  // 15
```

### 解引用（取出引用指向的值）
```rust
let x = 5;
let r = &x;      // r 是 x 的引用

println!("{}", r);   // 直接打印引用，OK
println!("{}", *r);  // 解引用，取出值 5

// 修改可变引用指向的值
let mut y = 10;
let r2 = &mut y;
*r2 = 20;  // 通过引用修改 y
println!("{}", y);  // 20
```

**什么时候要用 `*`？**

```rust
fn increment(n: &mut i32) {
    *n += 1;  // 必须解引用才能修改
}

let mut x = 5;
increment(&mut x);
println!("{}", x);  // 6
```

## `&` 和 `&mut` —— 不是位运算

**JS 里 `&` 是按位与，Rust 里是借用：**

```rust
// JS 的 &
let result = 5 & 3;  // 位运算，结果是 1

// Rust 的 &
let x = 5;
let r = &x;      // 不可变借用（引用）
let r2 = &mut x; // 可变借用
```

**Rust 也有位运算 `&`，但看上下文：**
```rust
let a = 5 & 3;        // 位运算，结果 1
let b = &some_var;    // 借用
```

## `::` 双冒号 —— 路径分隔符

```rust
// 访问模块/类型的东西
String::from("hello")      // String 类型的关联函数
std::io::stdin()           // 标准库路径
Vec::<i32>::new()          // 带泛型的调用（turbofish）

// 访问枚举变体
Option::Some(5)
Option::None
Result::Ok(data)
Result::Err(e)
```

## `->` 返回类型 —— 不是箭头函数

**JS 的 `=>` 是箭头函数，Rust 的 `->` 是返回类型：**

```javascript
// JS
const add = (a, b) => a + b;
```

```rust
// Rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

// Rust 的闭包用 |...|
let add = |a, b| a + b;
```

## `|x|` 闭包语法

**JS 用 `() =>`，Rust 用 `||`：**

```javascript
// JS
const double = (x) => x * 2;
[1,2,3].map(x => x * 2);
```

```rust
// Rust
let double = |x| x * 2;
vec![1,2,3].iter().map(|x| x * 2);

// 带类型标注
let add = |x: i32, y: i32| -> i32 { x + y };
```

## `;` 分号 —— 影响返回值！

**这是 Rust 最坑的地方之一：**

```rust
fn five() -> i32 {
    5      // ✅ 返回 5（表达式）
}

fn five_wrong() -> i32 {
    5;     // ❌ 编译错误！加了分号变成语句，返回 ()
}
```

**规则：**
- 没分号 = 表达式，有返回值
- 有分号 = 语句，返回 `()`（unit，类似 void）

```rust
let x = {
    let a = 1;
    let b = 2;
    a + b      // 最后没分号，整个块返回 3
};
println!("{}", x);  // 3
```

## `'a` 单引号 —— 生命周期，不是字符

**单引号在 Rust 里有两个意思：**

```rust
// 字符（和其他语言一样）
let c: char = 'A';
let emoji: char = '🦀';

// 生命周期标注（Rust 独有）
fn longest<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() { s1 } else { s2 }
}
```

**怎么区分？**
- `'字符'` → 字符
- `'标识符` → 生命周期（后面跟字母/数字）

## `?` 问号 —— 错误传播，不是三元运算符

**JS 的 `?:` 是三元运算符，Rust 的 `?` 是错误传播：**

```javascript
// JS 三元
const result = condition ? "yes" : "no";
```

```rust
// Rust 没有三元运算符！用 if 表达式
let result = if condition { "yes" } else { "no" };

// Rust 的 ? 是错误传播
fn read_file() -> Result<String, io::Error> {
    let content = fs::read_to_string("file.txt")?;  // 失败直接返回 Err
    Ok(content)
}
```

## `_` 下划线 —— 忽略值

```rust
// 忽略变量（不产生警告）
let _unused = 42;

// 模式匹配中忽略
match some_option {
    Some(x) => println!("{}", x),
    _ => println!("其他情况"),  // 通配符
}

// 忽略部分值
let (x, _, z) = (1, 2, 3);  // 只要 x 和 z

// 数字分隔（可读性）
let million = 1_000_000;
```

## `..` 和 `..=` 范围

```rust
// .. 不包含右边界
for i in 0..5 {
    println!("{}", i);  // 0, 1, 2, 3, 4
}

// ..= 包含右边界
for i in 0..=5 {
    println!("{}", i);  // 0, 1, 2, 3, 4, 5
}

// 切片
let arr = [0, 1, 2, 3, 4];
let slice = &arr[1..3];  // [1, 2]
```

## 总结速查表

| 符号 | 其他语言 | Rust |
|-----|---------|------|
| `*x` | 乘法 | 乘法 / 解引用 |
| `&x` | 按位与 | 借用 / 按位与 |
| `::` | PHP 静态方法 | 路径分隔符 |
| `->` | 无 | 返回类型 |
| `\|x\|` | 无 | 闭包 |
| `;` | 语句结束 | 语句结束 / 影响返回值 |
| `'a` | 字符 | 字符 / 生命周期 |
| `?` | 三元运算符 | 错误传播 |
| `_` | 无 | 忽略 / 通配符 |

---

**下节课**：Enum 实战场景
