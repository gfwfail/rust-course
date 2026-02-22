# 第 82 课：std::hash — 哈希的艺术

> 日期：2026-02-23

## 概述

上节课讲了比较（`Eq`），这节课讲哈希（`Hash`）。它们是一对好基友 —— HashMap 的 key 必须**同时**实现 `Hash + Eq`。

## Hash Trait

```rust
pub trait Hash {
    fn hash<H: Hasher>(&self, state: &mut H);
}
```

看起来有点奇怪？为什么不是 `fn hash(&self) -> u64`？

因为 Rust 要支持**流式哈希**：你可以分多次喂数据给 Hasher，最后才计算结果。

## 基本用法

### 派生实现（推荐）

```rust
use std::collections::HashMap;

#[derive(Hash, PartialEq, Eq)]
struct User {
    id: u64,
    name: String,
}

fn main() {
    let mut scores: HashMap<User, u32> = HashMap::new();
    
    let alice = User { id: 1, name: "Alice".into() };
    scores.insert(alice, 100);
}
```

**铁律**：如果两个值 `a == b`，那么它们的哈希值**必须相等**。

```
a == b  →  hash(a) == hash(b)  ✓
hash(a) == hash(b)  →  a == b  ✗（不一定）
```

## 手动实现 Hash

什么时候需要？当你只想用**部分字段**做哈希时。

```rust
use std::hash::{Hash, Hasher};

struct User {
    id: u64,
    name: String,      // 用于显示
    cache: Vec<u8>,    // 临时缓存，不参与比较
}

// ⚠️ 只用 id 做比较
impl PartialEq for User {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}
impl Eq for User {}

// ⚠️ 只用 id 做哈希（必须和 Eq 一致！）
impl Hash for User {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
    }
}
```

## Hasher Trait

`Hasher` 是哈希算法的抽象：

```rust
pub trait Hasher {
    fn finish(&self) -> u64;           // 获取最终结果
    fn write(&mut self, bytes: &[u8]); // 写入数据
}
```

标准库用的是 `DefaultHasher`（SipHash 1-3）：

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

fn calculate_hash<T: Hash>(t: &T) -> u64 {
    let mut s = DefaultHasher::new();
    t.hash(&mut s);
    s.finish()
}

fn main() {
    let hash1 = calculate_hash(&"hello");
    let hash2 = calculate_hash(&"hello");
    let hash3 = calculate_hash(&"world");
    
    assert_eq!(hash1, hash2); // 相同字符串，相同哈希
    println!("hello: {}", hash1);
    println!("world: {}", hash3);
}
```

## BuildHasher — 控制哈希算法

HashMap 有一个类型参数 `S: BuildHasher`：

```rust
pub struct HashMap<K, V, S = RandomState> { ... }
```

默认的 `RandomState` 每次程序运行会用**不同的种子**，防止哈希碰撞攻击。

### 自定义 BuildHasher

```rust
use std::collections::HashMap;
use std::hash::{BuildHasher, Hasher};

// 一个简单（但不安全）的哈希器
struct SimpleHasher(u64);

impl Hasher for SimpleHasher {
    fn finish(&self) -> u64 { self.0 }
    fn write(&mut self, bytes: &[u8]) {
        for &b in bytes {
            self.0 = self.0.wrapping_mul(31).wrapping_add(b as u64);
        }
    }
}

struct SimpleBuildHasher;

impl BuildHasher for SimpleBuildHasher {
    type Hasher = SimpleHasher;
    fn build_hasher(&self) -> SimpleHasher {
        SimpleHasher(0)
    }
}

fn main() {
    let mut map: HashMap<&str, i32, SimpleBuildHasher> 
        = HashMap::with_hasher(SimpleBuildHasher);
    map.insert("hello", 42);
}
```

## 实战：复合 Key

```rust
use std::collections::HashMap;

#[derive(Hash, PartialEq, Eq)]
struct CacheKey {
    user_id: u64,
    resource: String,
    version: u32,
}

fn main() {
    let mut cache: HashMap<CacheKey, Vec<u8>> = HashMap::new();
    
    let key = CacheKey {
        user_id: 1,
        resource: "avatar".into(),
        version: 2,
    };
    
    cache.insert(key, vec![0xFF, 0xD8, 0xFF]);
}
```

## 常见陷阱

### ❌ Hash 和 Eq 不一致

```rust
use std::hash::{Hash, Hasher};

struct Bad {
    id: u64,
    timestamp: u64,
}

impl PartialEq for Bad {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id  // 只比 id
    }
}
impl Eq for Bad {}

impl Hash for Bad {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
        self.timestamp.hash(state);  // ❌ 多哈希了 timestamp！
    }
}

// 两个 Bad 可能 id 相同但 timestamp 不同
// 结果：它们 Eq 相等，但 Hash 不同
// HashMap 会出 bug！
```

### ✅ 正确做法

```rust
impl Hash for Bad {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);  // ✅ 只哈希参与 Eq 的字段
    }
}
```

## f64 为什么不能做 HashMap Key？

```rust
// 编译错误！
// let mut map: HashMap<f64, String> = HashMap::new();
```

因为 `f64` 没实现 `Hash`（也没实现 `Eq`）。

原因：`NaN != NaN`，违反了等价关系。

### 解决方案

```rust
use std::collections::HashMap;
use std::hash::{Hash, Hasher};

#[derive(Clone, Copy)]
struct OrderedFloat(f64);

impl PartialEq for OrderedFloat {
    fn eq(&self, other: &Self) -> bool {
        self.0.to_bits() == other.0.to_bits()
    }
}
impl Eq for OrderedFloat {}

impl Hash for OrderedFloat {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.0.to_bits().hash(state);
    }
}

fn main() {
    let mut map: HashMap<OrderedFloat, &str> = HashMap::new();
    map.insert(OrderedFloat(3.14), "pi");
}
```

## 总结

| 概念 | 作用 |
|------|------|
| `Hash` | 计算哈希值 |
| `Hasher` | 哈希算法抽象 |
| `BuildHasher` | 创建 Hasher |

### 黄金法则

> **`a == b` 必须意味着 `hash(a) == hash(b)`**
> 参与 `Eq` 的字段必须全部参与 `Hash`，反之不一定。

### 什么时候用 derive？

- 所有字段都参与比较 → `#[derive(Hash, PartialEq, Eq)]`
- 只有部分字段参与 → 手动实现（注意保持一致性）

## 练习

实现一个 `Point` 结构体，可以作为 HashMap 的 key：

```rust
struct Point {
    x: i32,
    y: i32,
    label: String,  // 不参与比较，只是标签
}

// 两个 Point 只要 x, y 相同就算相等
```

### 参考答案

```rust
use std::collections::HashMap;
use std::hash::{Hash, Hasher};

struct Point {
    x: i32,
    y: i32,
    label: String,
}

impl PartialEq for Point {
    fn eq(&self, other: &Self) -> bool {
        self.x == other.x && self.y == other.y
    }
}
impl Eq for Point {}

impl Hash for Point {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.x.hash(state);
        self.y.hash(state);
        // 注意：不哈希 label！
    }
}

fn main() {
    let mut map: HashMap<Point, &str> = HashMap::new();
    
    let p1 = Point { x: 1, y: 2, label: "A".into() };
    let p2 = Point { x: 1, y: 2, label: "B".into() };
    
    map.insert(p1, "first");
    map.insert(p2, "second");  // 会覆盖 p1！
    
    assert_eq!(map.len(), 1);  // 只有一个 entry
}
```
