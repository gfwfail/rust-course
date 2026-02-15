# 第4课：生命周期 Lifetime

## 这玩意解决什么问题？

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;  // r 借用了 x
    }  // x 死了
    
    println!("{}", r);  // ❌ r 指向的东西已经没了！
}
```

这叫**悬垂引用**（dangling reference）。C/C++ 里这种 bug 能跑起来，然后随机崩溃。

Rust 编译器直接拦住：
```
error: `x` does not live long enough
```

**生命周期就是编译器用来追踪"引用能活多久"的机制。**

## 大部分时候你不用写

编译器会自动推断：

```rust
fn first_word(s: &str) -> &str {
    // 编译器知道：返回的引用和输入的引用活一样久
    &s[0..5]
}
```

这叫**生命周期省略规则**（Lifetime Elision）。90% 的情况编译器自己搞定。

## 什么时候必须手写？

当编译器**搞不清楚**返回的引用跟哪个输入有关：

```rust
// ❌ 编译错误
fn longer(s1: &str, s2: &str) -> &str {
    if s1.len() > s2.len() { s1 } else { s2 }
}
```

编译器懵了：返回的引用到底跟 `s1` 活一样久，还是跟 `s2`？

**手动标注：**

```rust
fn longer<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() { s1 } else { s2 }
}
```

`'a` 读作 "lifetime a"，意思是：
- s1 和 s2 都至少活 `'a` 这么久
- 返回值也活 `'a` 这么久
- 编译器会确保调用时这个约束成立

## 怎么理解 `'a`？

**不是**让你指定"活多久"，而是告诉编译器**几个引用之间的关系**。

```rust
fn longer<'a>(s1: &'a str, s2: &'a str) -> &'a str
```

翻译成人话：
> "返回的引用，不会比 s1 和 s2 中**较短命**的那个活得更久"

## 结构体里存引用

```rust
// ❌ 编译错误
struct Request {
    url: &str,  // 这个引用能活多久？编译器不知道
}

// ✅ 告诉编译器
struct Request<'a> {
    url: &'a str,  // url 的生命周期是 'a
}

// 使用
fn handle(req: &Request) {
    println!("{}", req.url);
}

fn main() {
    let url = String::from("https://example.com");
    let req = Request { url: &url };  // req 的生命周期 ≤ url
    handle(&req);
}
```

**为什么结构体要标注？**

因为结构体可能被传来传去，编译器需要知道里面的引用什么时候会失效。

## Web 框架里常见

```rust
// 假设一个请求处理器
struct Context<'req> {
    body: &'req str,
    headers: &'req HashMap<String, String>,
}

fn parse_json<'a>(ctx: &'a Context) -> Result<Value, Error> {
    serde_json::from_str(ctx.body)
}
```

这里 `'req` 表示"请求生命周期"——请求存在期间，这些引用都有效。

## 特殊生命周期：`'static`

活得和程序一样久：

```rust
let s: &'static str = "hello";  // 字符串字面量永远有效

// 常量也是 'static
const MAX: &'static i32 = &100;
```

**什么时候用 `'static`？**
- 字符串字面量
- 全局配置
- 线程间传递时有时需要（因为线程可能比当前作用域活得久）

## 生命周期省略规则（面试爱考）

编译器自动推断的三条规则：

1. **每个引用参数得到自己的生命周期**
   ```rust
   fn foo(x: &str, y: &str)
   // 变成 fn foo<'a, 'b>(x: &'a str, y: &'b str)
   ```

2. **如果只有一个输入生命周期，输出用同一个**
   ```rust
   fn foo(x: &str) -> &str
   // 变成 fn foo<'a>(x: &'a str) -> &'a str
   ```

3. **如果有 `&self`，输出用 `self` 的生命周期**
   ```rust
   fn method(&self, x: &str) -> &str
   // 返回值跟 self 活一样久
   ```

## 总结

| 概念 | 一句话解释 |
|-----|-----------|
| 生命周期 | 编译器追踪引用有效期的机制 |
| `'a` | 一个生命周期标签，表示"某段时间" |
| 标注 | 告诉编译器多个引用之间的关系 |
| `'static` | 活到程序结束 |
| 省略规则 | 编译器自动推断，不用每次都写 |

**核心思想**：Rust 在编译期就保证"引用永远指向有效的数据"。

---

**下节课**：错误处理
