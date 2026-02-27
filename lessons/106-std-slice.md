# 第 106 课：std::slice — 切片方法大全

## 什么是切片？

```rust
let arr = [1, 2, 3, 4, 5];
let slice: &[i32] = &arr[1..4];  // [2, 3, 4]

// 类比其他语言：
// PHP:   array_slice($arr, 1, 3)
// JS:    arr.slice(1, 4)
// Rust:  &arr[1..4]
```

**核心理解**：
- `[T]` 是切片类型（unsized，不能直接使用）
- `&[T]` 是切片的引用（胖指针 = 指针 + 长度）
- 切片是连续内存的「视图」，不拥有数据

---

## 一、基础查询

```rust
let nums = &[10, 20, 30, 40, 50];

nums.len()          // 5
nums.is_empty()     // false
nums.first()        // Some(&10)
nums.last()         // Some(&50)

// 安全索引（越界返回 None）
nums.get(2)         // Some(&30)
nums.get(100)       // None

// 直接索引（越界 panic！）
nums[2]             // 30
nums[100]           // 💥 panic!
```

💡 **小技巧**：生产代码用 `get()`，调试/测试用 `[]`

---

## 二、分割切片

```rust
let nums = &[1, 2, 3, 4, 5];

// split_at — 从指定位置一分为二
let (left, right) = nums.split_at(2);
// left: [1, 2], right: [3, 4, 5]

// split_first / split_last — 头尾分离
let (first, rest) = nums.split_first().unwrap();
// first: &1, rest: [2, 3, 4, 5]

let (last, init) = nums.split_last().unwrap();
// last: &5, init: [1, 2, 3, 4]

// split — 按条件分割（类似 PHP explode）
let data = &[1, 0, 2, 0, 3];
let parts: Vec<_> = data.split(|&x| x == 0).collect();
// [[1], [2], [3]]
```

### 分块处理

```rust
let nums = &[1, 2, 3, 4, 5, 6, 7];

// chunks — 等分切块
for chunk in nums.chunks(3) {
    println!("{:?}", chunk);
}
// [1, 2, 3]
// [4, 5, 6]
// [7]  (最后一块可能不足 3 个)

// chunks_exact — 严格等分（丢弃余数）
for chunk in nums.chunks_exact(3) {
    println!("{:?}", chunk);
}
// [1, 2, 3]
// [4, 5, 6]
// (7 被丢弃)

// windows — 滑动窗口
for win in nums.windows(3) {
    println!("{:?}", win);
}
// [1, 2, 3] → [2, 3, 4] → [3, 4, 5] → ...
```

🔥 **实战场景**：处理分页数据用 `chunks`，计算移动平均用 `windows`

---

## 三、搜索与查找

```rust
let nums = &[3, 1, 4, 1, 5, 9, 2, 6];

// contains — 是否包含
nums.contains(&5)           // true
nums.contains(&100)         // false

// starts_with / ends_with — 前后缀匹配
nums.starts_with(&[3, 1])   // true
nums.ends_with(&[2, 6])     // true

// 查找位置
let fruits = &["apple", "banana", "cherry"];
fruits.iter().position(|&x| x == "banana")  // Some(1)
```

### 二分查找（需要已排序！）

```rust
let sorted = &[1, 2, 3, 4, 5, 6, 7, 8, 9];

// binary_search — O(log n) 查找
match sorted.binary_search(&5) {
    Ok(index) => println!("找到，在位置 {index}"),  // 4
    Err(pos) => println!("没找到，应插入位置 {pos}"),
}

// 找不到时返回「应该插入的位置」
match sorted.binary_search(&0) {
    Err(pos) => println!("0 应该插在位置 {pos}"),  // 0
    _ => {}
}

// 自定义比较函数
let items = &[(1, "a"), (3, "b"), (5, "c")];
items.binary_search_by(|probe| probe.0.cmp(&3))  // Ok(1)
items.binary_search_by_key(&5, |item| item.0)    // Ok(2)
```

💡 PHP 没有内置二分查找，Rust 标准库自带！

---

## 四、排序（需要 &mut [T]）

```rust
let mut nums = vec![3, 1, 4, 1, 5, 9, 2, 6];
let slice: &mut [i32] = &mut nums;

// sort — 稳定排序
slice.sort();
// [1, 1, 2, 3, 4, 5, 6, 9]

// sort_unstable — 更快，但不稳定（相等元素顺序可能变）
let mut v = vec![5, 2, 8, 1];
v.sort_unstable();

// 自定义排序
let mut words = vec!["banana", "apple", "cherry"];
words.sort_by(|a, b| a.len().cmp(&b.len()));  // 按长度
// ["apple", "banana", "cherry"]

// sort_by_key — 更简洁
let mut items = vec![("z", 3), ("a", 1), ("m", 2)];
items.sort_by_key(|item| item.1);  // 按第二个元素排
// [("a", 1), ("m", 2), ("z", 3)]

// 倒序
let mut v = vec![1, 2, 3];
v.sort_by(|a, b| b.cmp(a));  // [3, 2, 1]
// 或者
v.sort();
v.reverse();
```

⚠️ **稳定 vs 不稳定**：
- 稳定：相等元素保持原有顺序
- 不稳定：更快，但顺序可能变

---

## 五、原地修改

```rust
let mut nums = vec![1, 2, 3, 4, 5];
let slice: &mut [i32] = &mut nums;

// reverse — 反转
slice.reverse();  // [5, 4, 3, 2, 1]

// rotate_left / rotate_right — 旋转
let mut v = vec![1, 2, 3, 4, 5];
v.rotate_left(2);   // [3, 4, 5, 1, 2]

let mut v = vec![1, 2, 3, 4, 5];
v.rotate_right(2);  // [4, 5, 1, 2, 3]

// swap — 交换两个位置
let mut v = vec![1, 2, 3];
v.swap(0, 2);  // [3, 2, 1]

// fill — 填充相同值
let mut v = vec![0, 0, 0, 0];
v.fill(42);  // [42, 42, 42, 42]

// fill_with — 用闭包填充
let mut v = vec![0; 5];
let mut counter = 0;
v.fill_with(|| {
    counter += 1;
    counter * 10
});
// [10, 20, 30, 40, 50]

// copy_from_slice — 从另一个切片复制
let src = [1, 2, 3];
let mut dst = [0, 0, 0];
dst.copy_from_slice(&src);  // dst 变成 [1, 2, 3]
```

---

## 六、迭代器方法

```rust
let nums = &[1, 2, 3, 4, 5];

// iter — 借用迭代
for n in nums.iter() {
    println!("{n}");  // n: &i32
}

// iter_mut — 可变借用迭代
let mut v = vec![1, 2, 3];
for n in v.iter_mut() {
    *n *= 2;  // 原地修改
}
// v: [2, 4, 6]

// 配合 enumerate
for (i, val) in nums.iter().enumerate() {
    println!("[{i}] = {val}");
}
```

### 特殊迭代器

```rust
let nums = &[10, 20, 30];

// rchunks — 从尾部开始分块
let v: Vec<_> = nums.rchunks(2).collect();
// [[20, 30], [10]]

// 同时迭代两个切片
let a = &[1, 2, 3];
let b = &[4, 5, 6];
for (x, y) in a.iter().zip(b.iter()) {
    println!("{x} + {y} = {}", x + y);
}
// 1 + 4 = 5
// 2 + 5 = 7
// 3 + 6 = 9
```

---

## 七、实战示例

### 找出 Top 3

```rust
fn top_n(nums: &[i32], n: usize) -> Vec<i32> {
    let mut sorted = nums.to_vec();
    sorted.sort_by(|a, b| b.cmp(a));  // 降序
    sorted.into_iter().take(n).collect()
}

let scores = &[85, 92, 78, 95, 88, 91];
let top3 = top_n(scores, 3);  // [95, 92, 91]
```

### 批量处理数据

```rust
fn process_in_batches(data: &[u32], batch_size: usize) {
    for (i, batch) in data.chunks(batch_size).enumerate() {
        println!("Batch {}: {:?}", i + 1, batch);
        // 这里处理每个批次...
    }
}

process_in_batches(&[1, 2, 3, 4, 5, 6, 7], 3);
// Batch 1: [1, 2, 3]
// Batch 2: [4, 5, 6]
// Batch 3: [7]
```

### 计算移动平均

```rust
fn moving_average(prices: &[f64], window: usize) -> Vec<f64> {
    prices
        .windows(window)
        .map(|w| w.iter().sum::<f64>() / window as f64)
        .collect()
}

let prices = &[100.0, 102.0, 104.0, 103.0, 105.0];
let ma = moving_average(prices, 3);
// [102.0, 103.0, 104.0]
```

---

## 📝 本课总结

| 类别 | 方法 | 用途 |
|-----|------|------|
| 查询 | `len`, `first`, `last`, `get` | 基础信息 |
| 分割 | `split_at`, `chunks`, `windows` | 拆分处理 |
| 搜索 | `contains`, `binary_search` | 查找元素 |
| 排序 | `sort`, `sort_by`, `sort_by_key` | 排序 |
| 修改 | `reverse`, `rotate`, `fill`, `swap` | 原地改变 |
| 迭代 | `iter`, `iter_mut`, `enumerate` | 遍历 |

### 切片 vs 数组 vs Vec

```
[T; N]  — 数组，大小固定，栈上
[T]     — 切片类型（unsized）
&[T]    — 切片引用，连续内存的视图
Vec<T>  — 动态数组，堆上，可增长
```

### 🔑 记忆技巧

- `chunks(n)` = 切成 n 个一份
- `windows(n)` = n 宽的滑动窗口
- `split_at(i)` = 从 i 处一刀切
- `binary_search` = 二分查找（前提：已排序！）

---

*下节课预告：`std::vec` — Vec 独有方法（与切片的区别）*
