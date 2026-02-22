# 第 81 课：std::cmp — 比较与排序

> 日期：2026-02-22

## 概述

`std::cmp` 模块定义了 Rust 中比较相关的核心 traits，是排序、查找、去重等操作的基础。

## 四大比较 Trait

```
Eq ← PartialEq
 ↑       ↑
Ord ← PartialOrd
```

| Trait | 作用 | 必须实现 |
|-------|------|----------|
| `PartialEq` | 判断相等 `==` `!=` | - |
| `Eq` | 完全等价关系 | `PartialEq` |
| `PartialOrd` | 部分比较 `<` `>` `<=` `>=` | `PartialEq` |
| `Ord` | 全序关系 | `PartialOrd` + `Eq` |

## PartialEq — 部分等价

```rust
// 自动派生（大部分情况够用）
#[derive(PartialEq)]
struct User {
    id: u64,
    name: String,
}

// 只按 id 比较（忽略 name）
impl PartialEq for User {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

let a = User { id: 1, name: "Alice".into() };
let b = User { id: 1, name: "Bob".into() };
assert!(a == b); // true，只比较 id
```

### 为什么叫 "Partial"？

因为有些类型**不满足自反性**：`x == x` 不一定成立。

```rust
let nan = f64::NAN;
assert!(nan != nan); // NaN 不等于自己！
```

所以 `f64` 只实现了 `PartialEq`，没实现 `Eq`。

## Eq — 完全等价

```rust
#[derive(PartialEq, Eq)]
struct UserId(u64);
```

`Eq` 是一个**标记 trait**（marker trait），没有方法，只是声明：

> "这个类型的相等关系是**自反的**、**对称的**、**传递的**"

用途：`HashMap` 的 key 必须实现 `Eq`：

```rust
use std::collections::HashMap;

// 编译通过
let mut map: HashMap<UserId, String> = HashMap::new();

// 编译失败！f64 没有 Eq
// let mut bad: HashMap<f64, String> = HashMap::new();
```

## PartialOrd — 部分排序

```rust
use std::cmp::Ordering;

#[derive(PartialEq)]
struct Score(f64);

impl PartialOrd for Score {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        self.0.partial_cmp(&other.0)
    }
}

let a = Score(85.0);
let b = Score(90.0);
assert!(a < b);
```

返回 `Option<Ordering>` 是因为比较**可能失败**（比如 NaN）：

```rust
let nan = f64::NAN;
assert_eq!(nan.partial_cmp(&1.0), None); // 无法比较
```

`Ordering` 是一个枚举：

```rust
pub enum Ordering {
    Less,    // 小于
    Equal,   // 等于
    Greater, // 大于
}
```

## Ord — 全序关系

```rust
use std::cmp::Ordering;

#[derive(PartialEq, Eq, PartialOrd)]
struct Priority(u8);

impl Ord for Priority {
    fn cmp(&self, other: &Self) -> Ordering {
        self.0.cmp(&other.0)
    }
}
```

`Ord` 要求**任意两个值都能比较**，返回 `Ordering`（不是 `Option`）。

用途：**排序**需要 `Ord`：

```rust
let mut items = vec![Priority(3), Priority(1), Priority(2)];
items.sort(); // 需要 Ord
// [Priority(1), Priority(2), Priority(3)]
```

## 实战：自定义排序

### 场景：用户按积分降序，积分相同按 ID 升序

```rust
use std::cmp::Ordering;

#[derive(Debug, PartialEq, Eq)]
struct User {
    id: u64,
    score: u64,
}

impl Ord for User {
    fn cmp(&self, other: &Self) -> Ordering {
        // 先按积分降序
        other.score.cmp(&self.score)
            // 积分相同，按 ID 升序
            .then(self.id.cmp(&other.id))
    }
}

impl PartialOrd for User {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

fn main() {
    let mut users = vec![
        User { id: 3, score: 100 },
        User { id: 1, score: 200 },
        User { id: 2, score: 100 },
    ];
    
    users.sort();
    
    for u in &users {
        println!("{:?}", u);
    }
    // User { id: 1, score: 200 }
    // User { id: 2, score: 100 }
    // User { id: 3, score: 100 }
}
```

### 关键方法：Ordering::then

```rust
// 链式比较：先比 a，相等再比 b
a.cmp(&b).then(c.cmp(&d))
```

## 辅助函数

`std::cmp` 还提供几个常用函数：

```rust
use std::cmp::{min, max, min_by, max_by};

// 取最小/最大
assert_eq!(min(3, 5), 3);
assert_eq!(max(3, 5), 5);

// 自定义比较
let a = "hello";
let b = "hi";
assert_eq!(min_by(a, b, |x, y| x.len().cmp(&y.len())), "hi");
```

### Reverse — 反转排序

```rust
use std::cmp::Reverse;

let mut nums = vec![1, 3, 2];
nums.sort_by_key(|&x| Reverse(x));
// [3, 2, 1]
```

## 总结

| Trait | 返回值 | 用途 |
|-------|--------|------|
| `PartialEq` | `bool` | `==` `!=` |
| `Eq` | (标记) | HashMap key |
| `PartialOrd` | `Option<Ordering>` | `<` `>` |
| `Ord` | `Ordering` | `sort()` |

### 实现顺序

```
PartialEq → Eq
    ↓
PartialOrd → Ord
```

### 记忆口诀

> **Partial = 可能失败（Option/bool）**
> **Full = 必定成功（Ordering/marker）**

## 练习

给这个结构实现完整的比较 traits：

```rust
struct Version {
    major: u32,
    minor: u32,
    patch: u32,
}

// 比较规则：major > minor > patch
```

### 参考答案

```rust
use std::cmp::Ordering;

#[derive(Debug, PartialEq, Eq)]
struct Version {
    major: u32,
    minor: u32,
    patch: u32,
}

impl Ord for Version {
    fn cmp(&self, other: &Self) -> Ordering {
        self.major.cmp(&other.major)
            .then(self.minor.cmp(&other.minor))
            .then(self.patch.cmp(&other.patch))
    }
}

impl PartialOrd for Version {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

fn main() {
    let v1 = Version { major: 1, minor: 2, patch: 3 };
    let v2 = Version { major: 1, minor: 2, patch: 4 };
    let v3 = Version { major: 2, minor: 0, patch: 0 };
    
    assert!(v1 < v2);
    assert!(v2 < v3);
    
    let mut versions = vec![v3, v1, v2];
    versions.sort();
    // [v1, v2, v3]
}
```
