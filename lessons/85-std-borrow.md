# 第 85 课：std::borrow — Borrow, BorrowMut 和 ToOwned

> 日期：2026-02-23  
> 主题：标准库 borrow 模块深入

---

## 🤔 为什么需要 Borrow trait？

先看一个真实问题：

```rust
use std::collections::HashMap;

let mut map: HashMap<String, i32> = HashMap::new();
map.insert("hello".to_string(), 42);

// 问题：查找时必须传 &String 吗？
let value = map.get(&"hello".to_string()); // 这样写太蠢了！

// 能不能直接传 &str？
let value = map.get("hello"); // ✅ 居然可以！
```

`HashMap<String, V>` 的 key 是 `String`，但 `get()` 却能接受 `&str`。这是怎么做到的？

答案就是 **Borrow trait**。

---

## 📦 Borrow trait 详解

```rust
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}
```

`Borrow<T>` 的含义是：**我可以被借用为 `&T`**。

标准库为 `String` 实现了 `Borrow<str>`：

```rust
impl Borrow<str> for String {
    fn borrow(&self) -> &str {
        &self[..]
    }
}
```

所以 `String` 可以被"看作" `&str` 来使用。

---

## 🔑 Borrow vs AsRef 的区别

你可能会问：这和 `AsRef<T>` 有什么区别？

```rust
// AsRef — 轻量级引用转换
pub trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

// Borrow — 语义上的"借用等价"
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}
```

**关键区别：Borrow 有额外的语义保证！**

`Borrow` 要求：
1. **Hash 等价**：如果 `x.borrow() == y`，那么 `hash(x) == hash(y)`
2. **Eq 等价**：`x.borrow() == y.borrow()` ↔ `x == y`
3. **Ord 等价**：比较结果一致

这就是为什么 HashMap/HashSet 用 `Borrow` 而不是 `AsRef`！

```rust
use std::collections::HashSet;

fn contains_key<Q>(set: &HashSet<String>, key: &Q) -> bool
where
    String: Borrow<Q>,
    Q: Hash + Eq + ?Sized,
{
    set.contains(key)
}

let set: HashSet<String> = ["foo".to_string()].into();
// 可以用 &str 查找，因为 String: Borrow<str>
// 且 str 和 String 的 Hash/Eq 行为一致
assert!(contains_key(&set, "foo"));
```

---

## ✏️ BorrowMut — 可变借用

```rust
pub trait BorrowMut<Borrowed: ?Sized>: Borrow<Borrowed> {
    fn borrow_mut(&mut self) -> &mut Borrowed;
}
```

用法和 `Borrow` 类似，但返回可变引用：

```rust
use std::borrow::BorrowMut;

fn clear_string<T: BorrowMut<str>>(s: &mut T) {
    let borrowed: &mut str = s.borrow_mut();
    // str 没有 clear 方法，但可以做其他操作
    println!("长度: {}", borrowed.len());
}

let mut s = String::from("hello");
clear_string(&mut s);
```

---

## 🔄 ToOwned — 从借用到拥有

`Clone` 的问题是：`&str` 没法 clone 成 `String`（类型不一样）。

`ToOwned` 解决了这个问题：

```rust
pub trait ToOwned {
    type Owned: Borrow<Self>;
    
    fn to_owned(&self) -> Self::Owned;
}
```

标准库的实现：

```rust
impl ToOwned for str {
    type Owned = String;
    
    fn to_owned(&self) -> String {
        self.to_string()
    }
}

impl ToOwned for [T] {
    type Owned = Vec<T>;
    
    fn to_owned(&self) -> Vec<T> {
        self.to_vec()
    }
}
```

使用示例：

```rust
use std::borrow::ToOwned;

let s: &str = "hello";
let owned: String = s.to_owned(); // &str → String

let arr: &[i32] = &[1, 2, 3];
let owned: Vec<i32> = arr.to_owned(); // &[i32] → Vec<i32>
```

---

## 🐄 Cow 与 Borrow/ToOwned

还记得第 72 课的 `Cow` 吗？它就是建立在 `ToOwned` 之上的：

```rust
pub enum Cow<'a, B: ?Sized + ToOwned> {
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
```

`Cow<str>` 内部要么是 `&str`（借用），要么是 `String`（拥有）。能这样做就是因为 `str: ToOwned<Owned = String>`。

---

## 🛠️ 实战：设计灵活的 API

用 `Borrow` 让你的函数更灵活：

```rust
use std::borrow::Borrow;
use std::collections::HashMap;

struct Cache {
    data: HashMap<String, String>,
}

impl Cache {
    // 不好的设计：只能传 &String
    fn get_bad(&self, key: &String) -> Option<&String> {
        self.data.get(key)
    }
    
    // 好的设计：能传 &str 或 &String
    fn get_good<K>(&self, key: &K) -> Option<&String>
    where
        K: Borrow<str> + ?Sized,
    {
        self.data.get(key.borrow())
    }
}

let cache = Cache { data: HashMap::new() };
// cache.get_bad("key");  // ❌ 编译错误
cache.get_good("key");    // ✅ 可以传 &str
cache.get_good(&String::from("key")); // ✅ 也可以传 &String
```

---

## 📋 常见 Borrow 实现

| 类型 | Borrow<T> |
|------|-----------|
| `String` | `str` |
| `Vec<T>` | `[T]` |
| `Box<T>` | `T` |
| `Arc<T>` | `T` |
| `Rc<T>` | `T` |
| `PathBuf` | `Path` |
| `OsString` | `OsStr` |

---

## 🎯 总结

| Trait | 用途 | 特点 |
|-------|------|------|
| `Borrow<T>` | 借用为 T | 保证 Hash/Eq/Ord 等价 |
| `BorrowMut<T>` | 可变借用为 T | Borrow 的可变版本 |
| `ToOwned` | 借用变拥有 | 泛化的 Clone |
| `AsRef<T>` | 轻量引用转换 | 无语义保证 |

**使用建议**：
- 设计 HashMap key 查找 API → 用 `Borrow`
- 普通引用转换 → 用 `AsRef`
- 需要 `&T → Owned` → 用 `ToOwned`

---

*Rust 学习小组 · 第 85 课*
