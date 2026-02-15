# 第5课：错误处理

## Rust 不用异常，用返回值

你写 JS/PHP 习惯这样：

```javascript
try {
    const user = await getUser(id);
    const orders = await getOrders(user);
} catch (e) {
    console.log("出错了:", e);
}
```

**Rust 没有 try-catch**。错误就是普通的返回值。

## Option：可能有值，可能没有

```rust
// 类似 JS 的 nullable，但必须显式处理
enum Option<T> {
    Some(T),  // 有值
    None,     // 没值
}
```

**实战场景：查找元素**

```rust
let numbers = vec![1, 2, 3];
let found = numbers.get(10);  // 越界了

// JS 会返回 undefined，然后后面某处爆炸
// Rust 返回 Option，逼你处理

match found {
    Some(n) => println!("找到: {}", n),
    None => println!("没找到"),
}

// 或者简写
if let Some(n) = found {
    println!("找到: {}", n);
}
```

## Result：可能成功，可能失败

```rust
enum Result<T, E> {
    Ok(T),   // 成功，带返回值
    Err(E),  // 失败，带错误信息
}
```

**实战场景：读文件**

```rust
use std::fs;

fn read_config() -> Result<String, std::io::Error> {
    fs::read_to_string("config.json")
}

fn main() {
    match read_config() {
        Ok(content) => println!("配置: {}", content),
        Err(e) => println!("读取失败: {}", e),
    }
}
```

## `?` 操作符：错误传播神器

不想每个地方都 match？用 `?`：

```rust
fn get_user_name(id: u32) -> Result<String, DbError> {
    let conn = connect_db()?;      // 失败直接返回 Err
    let user = conn.find_user(id)?; // 失败直接返回 Err
    Ok(user.name)                   // 成功返回 Ok
}
```

`?` 的意思：
- 如果是 `Ok(v)`：解开，继续执行
- 如果是 `Err(e)`：直接 return Err(e)

**等价于：**
```rust
let conn = match connect_db() {
    Ok(c) => c,
    Err(e) => return Err(e),
};
```

但是 `?` 一行搞定。

## 对比其他语言

| 场景 | JS/PHP | Rust |
|-----|--------|------|
| 空值 | `null`/`undefined` 到处炸 | `Option`，必须处理 |
| 异常 | `throw` 可能被忘记 catch | `Result`，不处理编译不过 |
| 错误传播 | try-catch 一大坨 | `?` 一个字符 |

## 常用的快捷方法

### unwrap()：我确定有值（否则 panic）

```rust
let num: Option<i32> = Some(5);
let value = num.unwrap();  // 直接拿 5

let none: Option<i32> = None;
none.unwrap();  // 💥 panic!
```

**什么时候用？**
- 写 demo / 快速原型
- 你**逻辑上确定**不会是 None/Err
- 测试代码

**生产代码少用**，crash 了就 crash 了。

### expect()：unwrap 带自定义错误信息

```rust
let config = fs::read_to_string("config.json")
    .expect("配置文件必须存在");
```

比 unwrap 好一点，至少 panic 时知道为什么。

### unwrap_or()：给默认值

```rust
let port = env::var("PORT")
    .unwrap_or("8080".to_string());  // 没有就用 8080
```

### map() / and_then()：链式处理

```rust
// Option 版
let length = Some("hello")
    .map(|s| s.len());  // Some(5)

// Result 版
let content = fs::read_to_string("data.txt")
    .map(|s| s.to_uppercase());  // 成功就转大写
```

## 实战：Web 处理链路

```rust
fn handle_request(req: Request) -> Result<Response, ApiError> {
    let token = req.header("Authorization")
        .ok_or(ApiError::Unauthorized)?;  // Option → Result
    
    let user = validate_token(token)?;
    let data = fetch_user_data(user.id)?;
    
    Ok(Response::json(data))
}
```

一串 `?`，任何一步失败就直接返回错误。干净利落。

## 为什么不用异常？

1. **显式**：看函数签名就知道会不会出错
2. **强制处理**：忘记处理？编译不过
3. **零成本**：没有 try-catch 的运行时开销
4. **可组合**：`map`、`and_then`、`?` 链式操作

---

**下节课**：Rust 设计模式
