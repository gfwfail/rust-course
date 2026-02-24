# 第 99 课：std::collections — 标准库集合类型全览

之前我们单独讲过 `Vec` 和 `HashMap`，今天系统梳理一下 Rust 标准库提供的所有集合类型，理解它们的适用场景和性能特点。

---

## 📦 集合类型概览

```rust
use std::collections::{
    // 序列
    Vec,        // 动态数组（最常用）
    VecDeque,   // 双端队列
    LinkedList, // 双向链表
    
    // 映射
    HashMap,    // 哈希表
    BTreeMap,   // B 树映射（有序）
    
    // 集合
    HashSet,    // 哈希集合
    BTreeSet,   // B 树集合（有序）
    
    // 特殊用途
    BinaryHeap, // 二叉堆（优先队列）
};
```

---

## 🔢 序列类型对比

### Vec — 动态数组（90% 的场景用它）

```rust
let mut v = Vec::new();
v.push(1);
v.push(2);

// 随机访问 O(1)
let first = v[0];

// 尾部操作 O(1)
v.push(3);
v.pop();

// 中间插入/删除 O(n) — 慢！
v.insert(0, 100);  // 所有元素右移
v.remove(0);       // 所有元素左移
```

**适用场景**：需要随机访问、主要在尾部增删

---

### VecDeque — 双端队列

```rust
use std::collections::VecDeque;

let mut deque = VecDeque::new();

// 两端操作都是 O(1)
deque.push_back(1);   // [1]
deque.push_front(0);  // [0, 1]
deque.push_back(2);   // [0, 1, 2]

deque.pop_front();    // Some(0), 剩 [1, 2]
deque.pop_back();     // Some(2), 剩 [1]

// 随机访问也是 O(1)
let first = deque[0];

// 底层是环形缓冲区
println!("{:?}", deque);  // [1]
```

**适用场景**：需要在两端频繁增删（如实现队列、滑动窗口）

```rust
// 实现固定大小的滑动窗口
struct SlidingWindow {
    data: VecDeque<i32>,
    max_size: usize,
}

impl SlidingWindow {
    fn push(&mut self, val: i32) {
        if self.data.len() >= self.max_size {
            self.data.pop_front();  // 移除最老的
        }
        self.data.push_back(val);
    }
}
```

---

### LinkedList — 双向链表（很少用）

```rust
use std::collections::LinkedList;

let mut list = LinkedList::new();
list.push_back(1);
list.push_front(0);

// 任意位置插入/删除是 O(1)... 
// 但要先找到位置是 O(n)！
// 而且没有随机访问，遍历慢（缓存不友好）
```

**⚠️ 几乎不要用！** 现代 CPU 对数组（连续内存）有极好的缓存优化，`Vec` 在大多数场景比 `LinkedList` 快。

---

## 🗺️ 映射类型对比

### HashMap — 无序哈希表

```rust
use std::collections::HashMap;

let mut map = HashMap::new();
map.insert("apple", 3);
map.insert("banana", 2);

// 查找、插入、删除都是 O(1) 平均
if let Some(count) = map.get("apple") {
    println!("苹果有 {} 个", count);
}

// 遍历顺序不确定！
for (k, v) in &map {
    println!("{}: {}", k, v);
}
```

---

### BTreeMap — 有序 B 树映射

```rust
use std::collections::BTreeMap;

let mut map = BTreeMap::new();
map.insert("banana", 2);
map.insert("apple", 3);
map.insert("cherry", 1);

// 遍历是按 key 排序的！
for (k, v) in &map {
    println!("{}: {}", k, v);  
    // apple: 3
    // banana: 2
    // cherry: 1
}

// 支持范围查询
for (k, v) in map.range("a".."c") {
    println!("{}: {}", k, v);
    // apple: 3
    // banana: 2
}
```

### HashMap vs BTreeMap

| 操作 | HashMap | BTreeMap |
|------|---------|----------|
| 查找/插入/删除 | O(1) 平均 | O(log n) |
| 遍历顺序 | 无序 | 按 key 排序 |
| 范围查询 | ❌ | ✅ |
| Key 要求 | Hash + Eq | Ord |

**选择原则**：
- 需要排序/范围查询 → `BTreeMap`
- 其他情况 → `HashMap`（更快）

---

## 🎯 集合类型（只存 Key，不存 Value）

```rust
use std::collections::{HashSet, BTreeSet};

// HashSet — 无序，快
let mut set = HashSet::new();
set.insert("rust");
set.insert("go");
set.contains("rust");  // true, O(1)

// BTreeSet — 有序
let mut set = BTreeSet::new();
set.insert(3);
set.insert(1);
set.insert(2);
for x in &set {
    print!("{} ", x);  // 1 2 3 （有序）
}

// 集合运算
let a: HashSet<_> = [1, 2, 3].into_iter().collect();
let b: HashSet<_> = [2, 3, 4].into_iter().collect();

let union: HashSet<_> = a.union(&b).collect();        // {1,2,3,4}
let inter: HashSet<_> = a.intersection(&b).collect(); // {2,3}
let diff: HashSet<_> = a.difference(&b).collect();    // {1}
```

---

## 📊 BinaryHeap — 优先队列

```rust
use std::collections::BinaryHeap;

// 默认是最大堆
let mut heap = BinaryHeap::new();
heap.push(3);
heap.push(1);
heap.push(4);
heap.push(1);
heap.push(5);

// pop 总是返回最大值
while let Some(val) = heap.pop() {
    print!("{} ", val);  // 5 4 3 1 1
}

// 想要最小堆？用 Reverse
use std::cmp::Reverse;

let mut min_heap = BinaryHeap::new();
min_heap.push(Reverse(3));
min_heap.push(Reverse(1));
min_heap.push(Reverse(5));

// pop 返回最小值
while let Some(Reverse(val)) = min_heap.pop() {
    print!("{} ", val);  // 1 3 5
}
```

| 操作 | 复杂度 |
|------|--------|
| push | O(log n) |
| pop (取最大/最小) | O(log n) |
| peek (看最大/最小) | O(1) |

**适用场景**：任务调度、Top K 问题、Dijkstra 算法

---

## 💡 Entry API — 高效的条件插入

```rust
use std::collections::HashMap;

let mut map: HashMap<&str, i32> = HashMap::new();

// ❌ 低效写法：查两次
if !map.contains_key("key") {
    map.insert("key", 1);
}

// ✅ 高效写法：Entry API
map.entry("key").or_insert(0);  // 不存在则插入 0

// 统计词频
let text = "hello world hello rust";
let mut freq = HashMap::new();
for word in text.split_whitespace() {
    *freq.entry(word).or_insert(0) += 1;
}
// {"hello": 2, "world": 1, "rust": 1}

// or_insert_with 延迟计算
map.entry("key").or_insert_with(|| expensive_computation());
```

---

## 🧠 选型速查表

| 场景 | 推荐类型 |
|------|----------|
| 普通列表 | `Vec` |
| 队列 (FIFO) | `VecDeque` |
| 栈 (LIFO) | `Vec` |
| 键值存储 | `HashMap` |
| 有序键值存储 | `BTreeMap` |
| 去重 | `HashSet` |
| 有序去重 | `BTreeSet` |
| 优先队列 | `BinaryHeap` |
| 双向链表 | 几乎不用 `LinkedList` |

---

## 🎓 课后思考

1. 为什么 Rust 的 `LinkedList` 很少被使用？
2. `BTreeMap` 的 key 为什么需要实现 `Ord` 而不是 `Hash`？
3. 如何用 `BinaryHeap` 实现一个定时任务调度器？

---

*第 99 课完*
