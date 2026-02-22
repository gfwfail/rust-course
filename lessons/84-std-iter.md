# 第 84 课：std::iter — 迭代器的力量

> 日期：2026-02-23  
> 主题：迭代器核心概念、Iterator trait、常用方法

---

## 📍 为什么迭代器很重要？

迭代器是 Rust 的灵魂之一。几乎所有集合操作都基于迭代器：

```rust
// 这些你每天都在用
vec.iter().map(...).filter(...).collect()
```

理解迭代器，才能写出真正地道的 Rust 代码。

---

## 🎯 核心：Iterator trait

```rust
trait Iterator {
    type Item;  // 迭代产出的元素类型
    
    fn next(&mut self) -> Option<Self::Item>;
    
    // 还有 75+ 个默认方法...
}
```

**只需要实现一个方法 `next()`，就能获得所有迭代器能力！**

**类比 PHP：**
- `Iterator` trait ≈ PHP 的 `Iterator` 接口
- 但 Rust 的更强大，自带海量链式方法

---

## 🔧 三种迭代方式

```rust
let v = vec![1, 2, 3];

// 1. iter() — 借用 (&T)
for x in v.iter() {
    println!("{}", x);  // x 是 &i32
}
// v 还能用

// 2. iter_mut() — 可变借用 (&mut T)
let mut v2 = vec![1, 2, 3];
for x in v2.iter_mut() {
    *x *= 2;  // 原地修改
}
// v2 = [2, 4, 6]

// 3. into_iter() — 获取所有权 (T)
for x in v.into_iter() {
    println!("{}", x);  // x 是 i32
}
// v 已被消耗，不能再用！
```

**记忆口诀：**
- `iter()` = 借着看看
- `iter_mut()` = 借着改改  
- `into_iter()` = 拿走了

---

## 🚀 常用迭代器方法

### 转换类

```rust
let nums = vec![1, 2, 3, 4, 5];

// map — 转换每个元素
let doubled: Vec<_> = nums.iter()
    .map(|x| x * 2)
    .collect();
// [2, 4, 6, 8, 10]

// filter — 过滤元素
let evens: Vec<_> = nums.iter()
    .filter(|x| *x % 2 == 0)
    .collect();
// [2, 4]

// filter_map — 过滤 + 转换
let parsed: Vec<i32> = ["1", "two", "3"].iter()
    .filter_map(|s| s.parse().ok())
    .collect();
// [1, 3]
```

### 聚合类

```rust
let nums = vec![1, 2, 3, 4, 5];

// fold — 折叠求和
let sum = nums.iter().fold(0, |acc, x| acc + x);
// 15

// sum — 直接求和
let sum: i32 = nums.iter().sum();
// 15

// count — 计数
let count = nums.iter().count();
// 5

// any / all — 存在/全部满足
let has_even = nums.iter().any(|x| x % 2 == 0);  // true
let all_positive = nums.iter().all(|x| *x > 0);   // true
```

### 查找类

```rust
let nums = vec![1, 2, 3, 4, 5];

// find — 找第一个满足条件的
let first_even = nums.iter().find(|x| *x % 2 == 0);
// Some(&2)

// position — 找索引
let pos = nums.iter().position(|x| *x == 3);
// Some(2)

// max / min
let max = nums.iter().max();  // Some(&5)
let min = nums.iter().min();  // Some(&1)
```

---

## ⛓️ 链式调用的艺术

```rust
let users = vec![
    ("Alice", 30),
    ("Bob", 17),
    ("Charlie", 25),
    ("Diana", 15),
];

// 找出所有成年人的名字，按年龄排序
let adults: Vec<&str> = users.iter()
    .filter(|(_, age)| *age >= 18)    // 过滤成年人
    .map(|(name, _)| *name)           // 提取名字
    .collect();
// ["Alice", "Charlie"]
```

---

## 🔄 惰性求值

**Rust 迭代器是惰性的！**

```rust
let nums = vec![1, 2, 3];

// 这行什么都不做！只是构建了一个迭代器
let iter = nums.iter().map(|x| {
    println!("处理 {}", x);
    x * 2
});

// 直到 collect 才真正执行
let result: Vec<_> = iter.collect();
// 这时才打印：处理 1, 处理 2, 处理 3
```

**类比 Laravel：**
```php
// Laravel Collection 是即时求值
$result = collect([1,2,3])->map(fn($x) => $x * 2);

// Rust 迭代器是惰性的，类似 Laravel 的 LazyCollection
```

---

## 📦 collect 的魔法

`collect()` 可以收集成各种类型：

```rust
let chars = ['a', 'b', 'c'];

// 收集成 Vec
let v: Vec<char> = chars.iter().copied().collect();

// 收集成 String
let s: String = chars.iter().collect();

// 收集成 HashSet
use std::collections::HashSet;
let set: HashSet<char> = chars.iter().copied().collect();

// 收集成 Result
let nums = ["1", "2", "3"];
let parsed: Result<Vec<i32>, _> = nums.iter()
    .map(|s| s.parse())
    .collect();
// Ok([1, 2, 3])
```

---

## 🛠️ 自己实现 Iterator

```rust
struct Counter {
    current: u32,
    max: u32,
}

impl Counter {
    fn new(max: u32) -> Self {
        Counter { current: 0, max }
    }
}

impl Iterator for Counter {
    type Item = u32;
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.current < self.max {
            self.current += 1;
            Some(self.current)
        } else {
            None
        }
    }
}

fn main() {
    let counter = Counter::new(5);
    
    // 现在可以用所有迭代器方法了！
    let sum: u32 = counter.sum();
    println!("{}", sum);  // 15 (1+2+3+4+5)
}
```

---

## 💡 实用技巧

```rust
// enumerate — 带索引迭代
for (i, x) in vec![10, 20, 30].iter().enumerate() {
    println!("{}: {}", i, x);
}
// 0: 10
// 1: 20
// 2: 30

// zip — 并行迭代两个迭代器
let a = [1, 2, 3];
let b = ["one", "two", "three"];
for (num, name) in a.iter().zip(b.iter()) {
    println!("{} = {}", num, name);
}

// take / skip — 取前 N 个 / 跳过前 N 个
let first_two: Vec<_> = (1..10).take(2).collect();  // [1, 2]
let skip_two: Vec<_> = (1..5).skip(2).collect();    // [3, 4]

// chain — 连接两个迭代器
let combined: Vec<_> = [1, 2].iter()
    .chain([3, 4].iter())
    .collect();
// [1, 2, 3, 4]
```

---

## 📝 课后思考

1. **为什么 `iter()` 返回 `&T` 而不是 `T`？**
   - 避免所有权转移，让原集合可以继续使用

2. **`for x in v` 和 `for x in v.iter()` 有什么区别？**
   - `for x in v` 等价于 `for x in v.into_iter()`，会消耗 v
   - `for x in v.iter()` 只是借用

3. **为什么迭代器要惰性求值？**
   - 性能！不需要创建中间集合
   - 可以处理无限序列

---

## 🎯 下节预告

下节课讲 `std::iter` 的进阶用法：`Peekable`、`Fuse`、`Rev` 等适配器！

---

*迭代器是 Rust 的超能力，熟练掌握它！*
