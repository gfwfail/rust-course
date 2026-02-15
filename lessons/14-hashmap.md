# 第14课：HashMap

键值对存储，像字典一样——给一个 key，找到对应的 value。

## 创建 HashMap

```rust
use std::collections::HashMap;

// 方法一：new() + insert
let mut map = HashMap::new();
map.insert("apple", 3);
map.insert("banana", 5);

// 方法二：从元组数组 collect
let pairs = vec![("apple", 3), ("banana", 5)];
let map: HashMap<_, _> = pairs.into_iter().collect();
```

⚠️ 注意：`HashMap` 不在 prelude 里，必须 `use` 引入！

## 读取值

```rust
let mut scores = HashMap::new();
scores.insert("蓝队", 10);
scores.insert("红队", 50);

// get() 返回 Option<&V>
match scores.get("蓝队") {
    Some(score) => println!("蓝队得分：{}", score),
    None => println!("没有这个队"),
}

// copied() 把 Option<&i32> 变成 Option<i32>
let blue = scores.get("蓝队").copied().unwrap_or(0);
```

⚡ **小技巧**：`unwrap_or(默认值)` 处理 None 的情况

## 遍历 HashMap

```rust
for (key, value) in &scores {
    println!("{}: {}", key, value);
}
```

⚠️ 顺序不固定！HashMap 是无序的。

## 更新值

```rust
// 直接覆盖
scores.insert("蓝队", 25);

// 不存在才插入：entry + or_insert
scores.entry("绿队").or_insert(30);  // 没有就插入 30
scores.entry("蓝队").or_insert(99);  // 已存在，不变

// 基于旧值更新（词频统计神器）
let text = "hello world hello rust";

let mut word_count = HashMap::new();
for word in text.split_whitespace() {
    let count = word_count.entry(word).or_insert(0);
    *count += 1;  // count 是 &mut V，解引用后 +1
}
// {"hello": 2, "world": 1, "rust": 1}
```

## 所有权规则

```rust
let key = String::from("name");
let value = String::from("Rust");

let mut map = HashMap::new();
map.insert(key, value);

// ❌ key 和 value 已经移动进 map 了！
// println!("{}", key);  // 报错！
```

**解决办法**：
- 用引用作为 key/value（需要考虑生命周期）
- 用 `Clone` 类型
- 或者用 Copy 类型（如 i32）直接复制

## 常用方法

| 方法 | 作用 |
|------|------|
| `insert(k, v)` | 插入/覆盖 |
| `get(&k)` | 获取 Option<&V> |
| `remove(&k)` | 删除并返回 Option<V> |
| `contains_key(&k)` | 是否包含 key |
| `keys()` | 返回所有 key 的迭代器 |
| `values()` | 返回所有 value 的迭代器 |
| `len()` | 键值对数量 |
| `is_empty()` | 是否为空 |
| `clear()` | 清空 |

## 实战：统计字符出现次数

```rust
use std::collections::HashMap;

fn main() {
    let s = "abracadabra";
    let mut freq = HashMap::new();
    
    for c in s.chars() {
        *freq.entry(c).or_insert(0) += 1;
    }
    
    for (ch, count) in &freq {
        println!("'{}' 出现了 {} 次", ch, count);
    }
}
```

---

**下节课**：迭代器 Iterators
