# 第17课：Box 堆分配

Box 是最简单的智能指针，把数据放到堆上。

## 基本用法

```rust
fn main() {
    // 在堆上分配一个 i32
    let b = Box::new(5);
    println!("b = {}", b);  // 自动解引用
    
    // 离开作用域时自动释放堆内存
}
```

## 什么时候用 Box？

### 1. 递归类型（编译时大小未知）

```rust
// ❌ 编译失败：大小无限！
// enum List {
//     Cons(i32, List),
//     Nil,
// }

// ✅ Box 打破递归，指针大小固定
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let list = Cons(1, 
        Box::new(Cons(2, 
            Box::new(Cons(3, 
                Box::new(Nil))))));
}
```

### 2. 转移大数据所有权，避免拷贝

```rust
fn main() {
    let big_data = Box::new([0u8; 1_000_000]);
    let moved = big_data;  // 只移动指针，不复制 1MB 数据
}
```

### 3. Trait 对象

```rust
trait Animal {
    fn speak(&self);
}

struct Dog;
impl Animal for Dog {
    fn speak(&self) { println!("汪！"); }
}

fn main() {
    let animals: Vec<Box<dyn Animal>> = vec![
        Box::new(Dog),
    ];
}
```

## Deref 自动解引用

Box 实现了 `Deref`，可以像使用普通引用一样：

```rust
let b = Box::new(5);
let sum = *b + 10;  // 解引用取值
println!("{}", sum); // 15
```

## Drop 自动释放

```rust
{
    let b = Box::new(String::from("hello"));
    // 使用 b...
}  // b 离开作用域，堆内存自动释放
```

---

**下节课**：Rc 引用计数
