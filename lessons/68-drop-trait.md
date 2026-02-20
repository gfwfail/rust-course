# 第 68 课：Drop trait 与资源管理

> 日期：2026-02-21  
> 主题：Drop, std::mem::drop, RAII 模式

---

## 今天的问题

Rust 没有 GC，没有手动 `free()`。那资源（文件、网络连接、锁）是怎么释放的？

答案：**Drop trait + RAII**

---

## Drop trait 是什么？

```rust
pub trait Drop {
    fn drop(&mut self);
}
```

当一个值离开作用域时，Rust 自动调用它的 `drop()` 方法。

**你不需要手动调用它，编译器帮你做。**

---

## 最简单的例子

```rust
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("🗑️ 释放资源: {}", self.name);
    }
}

fn main() {
    let r = Resource { 
        name: String::from("数据库连接") 
    };
    println!("使用资源...");
    // r 在这里离开作用域
}

// 输出：
// 使用资源...
// 🗑️ 释放资源: 数据库连接
```

---

## Drop 的执行顺序

```rust
fn main() {
    let a = Resource { name: String::from("A") };
    let b = Resource { name: String::from("B") };
    let c = Resource { name: String::from("C") };
}

// 输出顺序（后进先出！）：
// 🗑️ 释放资源: C
// 🗑️ 释放资源: B
// 🗑️ 释放资源: A
```

**规则：变量按声明的逆序 drop（像栈一样）**

这很重要！确保依赖关系正确。

---

## 结构体字段的 Drop 顺序

```rust
struct Container {
    first: Resource,
    second: Resource,
}

// 字段按声明顺序 drop！
// 先 first，后 second
```

**结构体字段：按声明顺序 drop**
**局部变量：按声明逆序 drop**

---

## 提前释放：std::mem::drop()

不能直接调用 `.drop()`（编译器不允许）：

```rust
let r = Resource { ... };
r.drop();  // ❌ 编译错误！
```

用 `std::mem::drop()` 函数：

```rust
use std::mem::drop;

let r = Resource { name: String::from("连接") };
println!("开始使用");
drop(r);  // ✅ 提前释放
println!("已释放，继续其他工作");
// r 在这里已经无效，不会再 drop
```

---

## drop() 函数的真面目

```rust
// 标准库里的定义，简单到离谱
pub fn drop<T>(_x: T) {}
```

**就是个空函数！**

它接收所有权，函数结束时 `_x` 离开作用域，自动触发 Drop。

这就是 Rust 所有权系统的优雅之处。

---

## 实战场景：锁的自动释放

```rust
use std::sync::Mutex;

let data = Mutex::new(vec![1, 2, 3]);

{
    let mut guard = data.lock().unwrap();
    guard.push(4);
    // guard 离开作用域，锁自动释放
}

// 这里锁已经释放，其他线程可以获取
```

**不需要手动 `unlock()`！** 这就是 RAII。

---

## 实战场景：文件自动关闭

```rust
use std::fs::File;
use std::io::Write;

fn write_file() -> std::io::Result<()> {
    let mut file = File::create("test.txt")?;
    file.write_all(b"hello")?;
    // file 离开作用域，自动 close
    Ok(())
}
```

**不需要手动 `close()`！** 即使发生 panic，文件也会被关闭。

---

## Drop 与 Copy 不兼容

```rust
#[derive(Copy, Clone)]
struct Point(i32, i32);

impl Drop for Point {
    fn drop(&mut self) {} // ❌ 编译错误！
}
```

**Copy 类型不能有 Drop。**

原因：Copy 类型是按位复制的，如果有 Drop，每个副本都会执行 drop，造成双重释放。

---

## ManuallyDrop — 阻止自动 Drop

```rust
use std::mem::ManuallyDrop;

let data = ManuallyDrop::new(String::from("hello"));

// data 离开作用域时不会被 drop
// 需要手动处理（通常用于 unsafe 代码）
```

---

## 常见面试题

**Q: 为什么不能直接调用 `.drop()`？**

```rust
let s = String::from("hello");
s.drop();  // ❌ 为什么？
```

A: 因为编译器还是会在作用域结束时再调用一次 drop，导致 **double free**！

`std::mem::drop()` 通过**移动所有权**来解决：值被移走了，原变量失效，不会触发第二次 drop。

---

## 要点总结

| 概念 | 说明 |
|------|------|
| `Drop` trait | 定义析构行为，自动调用 |
| 执行时机 | 值离开作用域时 |
| 变量顺序 | 后声明的先 drop |
| 字段顺序 | 先声明的先 drop |
| `std::mem::drop()` | 提前释放（通过移动所有权） |
| RAII | 资源获取即初始化，离开即释放 |

---

## 对比 PHP

```php
class Resource {
    public function __destruct() {
        echo "释放资源\n";
    }
}

$r = new Resource();
unset($r);  // 手动触发析构
// 或等待 GC
```

Rust 的 Drop：
- **确定性**：离开作用域一定执行（PHP 依赖 GC）
- **编译时保证**：不会遗漏
- **顺序确定**：方便管理依赖

---

## 下节预告

**标准库的智能指针：Rc 与 Arc**

当一个值需要多个所有者时怎么办？

---

*课程笔记：性奴001*
