# 第15课：迭代器 Iterators

迭代器是 Rust 函数式编程的核心，配合闭包使用威力巨大。

## 什么是迭代器？

核心是 `Iterator` trait：

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

每次 `next()` 返回 `Some(值)` 或 `None`（结束）。

## 创建迭代器

```rust
let v = vec![1, 2, 3];

let iter1 = v.iter();       // 借用 &T
let iter2 = v.iter_mut();   // 可变借用 &mut T
let iter3 = v.into_iter();  // 获取所有权 T
```

**口诀**：
- `iter()` = 借着看
- `iter_mut()` = 借着改
- `into_iter()` = 拿走

## 适配器（Adapters）— 链式转换

适配器把一个迭代器变成另一个。**懒求值**：不消费就不执行！

```rust
let v = vec![1, 2, 3, 4, 5];

// map: 转换每个元素
let doubled: Vec<i32> = v.iter()
    .map(|x| x * 2)
    .collect();  // [2, 4, 6, 8, 10]

// filter: 保留符合条件的
let evens: Vec<&i32> = v.iter()
    .filter(|x| *x % 2 == 0)
    .collect();  // [&2, &4]

// 链式调用
let result: Vec<i32> = v.iter()
    .filter(|x| *x > 2)
    .map(|x| x * 10)
    .collect();  // [30, 40, 50]
```

## 常用适配器

```rust
let v = vec![1, 2, 3, 4, 5];

// take: 只取前n个
v.iter().take(2);  // 1, 2

// skip: 跳过前n个  
v.iter().skip(1);  // 2, 3, 4, 5

// enumerate: 带索引
for (i, val) in v.iter().enumerate() {
    println!("{}: {}", i, val);
}

// zip: 合并两个迭代器
let a = [1, 2, 3];
let b = ["a", "b", "c"];
let zipped: Vec<_> = a.iter().zip(b.iter()).collect();
// [(1, "a"), (2, "b"), (3, "c")]

// flatten: 展平嵌套
let nested = vec![vec![1, 2], vec![3, 4]];
let flat: Vec<_> = nested.into_iter().flatten().collect();
// [1, 2, 3, 4]

// rev: 反向
v.iter().rev();  // 5, 4, 3, 2, 1
```

## 消费者（Consumers）— 触发执行

消费者终止迭代，产生结果：

```rust
let v = vec![1, 2, 3, 4, 5];

// collect: 收集成集合
let doubled: Vec<i32> = v.iter().map(|x| x * 2).collect();

// sum / product
let total: i32 = v.iter().sum();      // 15
let prod: i32 = v.iter().product();   // 120

// count: 计数
let n = v.iter().filter(|x| *x > 2).count(); // 3

// any / all: 存在/全部
let has_even = v.iter().any(|x| x % 2 == 0); // true
let all_pos = v.iter().all(|x| *x > 0);      // true

// find: 找第一个
let found = v.iter().find(|x| *x > 3); // Some(&4)

// max / min
let biggest = v.iter().max();  // Some(&5)
let smallest = v.iter().min(); // Some(&1)
```

## fold — 归约操作

把所有元素「折叠」成一个值：

```rust
let v = vec![1, 2, 3, 4, 5];

// fold: 有初始值
let sum = v.iter().fold(0, |acc, x| acc + x);
//                       ^初始  ^累加器
// 结果: 15

// reduce: 第一个元素作初始值
let product = v.iter()
    .copied()
    .reduce(|a, b| a * b);  // Some(120)
```

## 为什么迭代器高效？

Rust 迭代器是**零成本抽象**：
- 懒求值：只在需要时计算
- 内联优化：链式调用展开成单次循环
- 无中间集合：map().filter() 不创建临时 Vec

```rust
// 这两个性能一样！
let sum: i32 = (0..1000)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .sum();
```

---

**下节课**：闭包 Closures
