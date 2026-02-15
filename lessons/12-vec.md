# 第12课：Vec 动态数组

`Vec<T>` 是 Rust 最常用的集合类型，可变长度数组，存在堆上。

## 创建 Vec

```rust
// 创建空的 Vec
let mut numbers: Vec<i32> = Vec::new();

// 用宏快速创建（推荐）
let fruits = vec!["苹果", "香蕉", "橙子"];
```

## 常用操作

```rust
let mut v = vec![1, 2, 3];

// 添加元素
v.push(4);          // [1, 2, 3, 4]

// 访问元素
let first = v[0];   // 索引访问，越界会 panic
let second = v.get(1); // 返回 Option<&T>，更安全

// 删除最后一个
let last = v.pop(); // 返回 Option<T>

// 长度
println!("长度: {}", v.len());

// 是否为空
println!("空的? {}", v.is_empty());
```

## 安全访问：get() vs 索引

```rust
let v = vec![10, 20, 30];

// 索引 - 越界会 panic！
// let x = v[100]; // 💥 程序崩溃

// get - 返回 Option，越界返回 None
match v.get(100) {
    Some(val) => println!("值: {}", val),
    None => println!("索引越界了！"),
}
```

## 遍历 Vec

```rust
let v = vec![1, 2, 3, 4, 5];

// 不可变遍历
for item in &v {
    println!("{}", item);
}

// 可变遍历（需要 &mut）
let mut v2 = vec![1, 2, 3];
for item in &mut v2 {
    *item += 10;  // 解引用后修改
}
// v2 = [11, 12, 13]
```

## Vec 与所有权

```rust
let v = vec![String::from("hello")];

// 移动所有权！v[0] 之后 v 就废了
// let s = v[0]; // ❌ 编译错误

// 借用才对
let s = &v[0]; // ✅ 借用
```

## 用枚举存不同类型

Vec 要求元素类型相同，但可以用枚举绕过：

```rust
enum Cell {
    Int(i32),
    Float(f64),
    Text(String),
}

let row = vec![
    Cell::Int(42),
    Cell::Float(3.14),
    Cell::Text(String::from("hello")),
];
```

## 常用方法速览

```rust
let mut v = vec![3, 1, 4, 1, 5];

v.len()              // 长度
v.is_empty()         // 是否为空
v.push(9)            // 尾部添加
v.pop()              // 尾部删除，返回 Option
v.insert(0, 2)       // 指定位置插入
v.remove(0)          // 指定位置删除
v.clear()            // 清空
v.contains(&1)       // 是否包含
v.sort()             // 排序
v.reverse()          // 反转
v.dedup()            // 去除连续重复（需先排序）
```

---

**下节课**：String 深入
