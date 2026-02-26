# 第 103 课：std::result — Result 组合子方法大全

## 为什么要学组合子？

```rust
// 新手写法：到处都是 match
let file = match File::open("config.json") {
    Ok(f) => f,
    Err(e) => return Err(e),
};
let content = match read_to_string(file) {
    Ok(s) => s,
    Err(e) => return Err(e),
};

// 老手写法：链式调用
let content = File::open("config.json")
    .and_then(read_to_string)?;
```

**组合子 = 链式调用的基础**

---

## 一、提取值的方法

### 1. unwrap 系列 — 简单粗暴

```rust
let ok: Result<i32, &str> = Ok(42);
let err: Result<i32, &str> = Err("boom");

// unwrap — 成功返回值，失败 panic
ok.unwrap()     // → 42
err.unwrap()    // → panic!

// unwrap_or — 失败返回默认值
err.unwrap_or(0)           // → 0
err.unwrap_or_default()    // → 0 (i32::default())

// unwrap_or_else — 失败时调用闭包
err.unwrap_or_else(|e| {
    eprintln!("错误: {}", e);
    -1
})  // → -1

// expect — 自定义 panic 信息
err.expect("配置文件读取失败")  // → panic with message
```

**PHP/JS 对比：**
```php
// PHP: 直接访问或 ?? 默认值
$value = $result ?? 0;

// Rust: 显式处理
let value = result.unwrap_or(0);
```

---

### 2. ok / err — 转换为 Option

```rust
let ok: Result<i32, &str> = Ok(42);
let err: Result<i32, &str> = Err("boom");

ok.ok()   // → Some(42)
ok.err()  // → None

err.ok()  // → None
err.err() // → Some("boom")
```

**用途：当你只关心成功值，不关心错误时**
```rust
// 获取配置，没有就用默认
let port = config.get("port")
    .ok()
    .and_then(|s| s.parse().ok())
    .unwrap_or(8080);
```

---

## 二、检查状态的方法

```rust
let ok: Result<i32, &str> = Ok(42);
let err: Result<i32, &str> = Err("boom");

// is_ok / is_err
ok.is_ok()         // → true
ok.is_err()        // → false

// is_ok_and / is_err_and (Rust 1.70+)
ok.is_ok_and(|x| x > 40)      // → true
ok.is_ok_and(|x| x > 50)      // → false
err.is_err_and(|e| e == "boom") // → true
```

**实战例子：**
```rust
// 检查文件是否存在且可读
if fs::metadata("config.json")
    .is_ok_and(|m| m.is_file() && !m.permissions().readonly()) 
{
    println!("配置文件存在且可写");
}
```

---

## 三、转换值的方法

### 1. map — 转换成功值

```rust
let ok: Result<i32, &str> = Ok(42);
let err: Result<i32, &str> = Err("boom");

// map — 转换 Ok 中的值
ok.map(|x| x * 2)     // → Ok(84)
err.map(|x| x * 2)    // → Err("boom") 不变

// 链式转换
"42".parse::<i32>()
    .map(|x| x * 2)
    .map(|x| x.to_string())
// → Ok("84")
```

### 2. map_err — 转换错误值

```rust
let err: Result<i32, &str> = Err("boom");

// 把 &str 错误转成自定义错误
err.map_err(|e| format!("解析失败: {}", e))
// → Err("解析失败: boom")

// 常见用法：统一错误类型
fn read_config() -> Result<Config, MyError> {
    let content = fs::read_to_string("config.json")
        .map_err(|e| MyError::Io(e))?;
    let config = serde_json::from_str(&content)
        .map_err(|e| MyError::Parse(e))?;
    Ok(config)
}
```

### 3. map_or / map_or_else — 有默认值的转换

```rust
let ok: Result<i32, &str> = Ok(42);
let err: Result<i32, &str> = Err("boom");

// map_or — Ok 转换，Err 返回默认值
ok.map_or(0, |x| x * 2)   // → 84
err.map_or(0, |x| x * 2)  // → 0

// map_or_else — 两边都用闭包
err.map_or_else(
    |e| { eprintln!("{}", e); -1 },  // Err 分支
    |x| x * 2                         // Ok 分支
)  // → -1
```

---

## 四、链式操作方法

### 1. and_then — 链接返回 Result 的操作

```rust
fn parse_port(s: &str) -> Result<u16, &'static str> {
    s.parse().map_err(|_| "无效端口")
}

fn validate_port(p: u16) -> Result<u16, &'static str> {
    if p > 1024 { Ok(p) } else { Err("端口必须大于1024") }
}

// 链式调用
"8080".to_string()
    .parse::<u16>()
    .map_err(|_| "解析失败")
    .and_then(validate_port)
// → Ok(8080)

"80".to_string()
    .parse::<u16>()
    .map_err(|_| "解析失败")
    .and_then(validate_port)
// → Err("端口必须大于1024")
```

**对比 map vs and_then：**
```rust
// map: 闭包返回普通值 T
result.map(|x| x * 2)

// and_then: 闭包返回 Result<T, E>
result.and_then(|x| validate(x))
```

**类比 JS Promise：**
```javascript
// JS
promise.then(x => x * 2)           // map
promise.then(x => fetchMore(x))    // and_then
```

### 2. or_else — Err 时尝试恢复

```rust
fn try_parse_int(s: &str) -> Result<i32, &str> {
    s.parse().map_err(|_| "not a number")
}

fn try_parse_hex(s: &str) -> Result<i32, &str> {
    i32::from_str_radix(s.trim_start_matches("0x"), 16)
        .map_err(|_| "not hex either")
}

// 先尝试十进制，失败了试十六进制
"0xff"
    .pipe(|s| try_parse_int(s))
    .or_else(|_| try_parse_hex("0xff"))
// → Ok(255)
```

---

## 五、引用与转换

### 1. as_ref / as_mut — 借用内部值

```rust
let mut ok: Result<String, &str> = Ok("hello".to_string());

// as_ref: Result<T, E> → Result<&T, &E>
ok.as_ref().map(|s| s.len())  // → Ok(5)，不消耗 ok

// as_mut: Result<T, E> → Result<&mut T, &mut E>
ok.as_mut().map(|s| s.push_str(" world"));
// ok 现在是 Ok("hello world")
```

### 2. transpose — Result<Option<T>> ↔ Option<Result<T>>

```rust
let x: Result<Option<i32>, &str> = Ok(Some(42));
let y: Option<Result<i32, &str>> = x.transpose();
// y = Some(Ok(42))

let x: Result<Option<i32>, &str> = Ok(None);
let y: Option<Result<i32, &str>> = x.transpose();
// y = None
```

**用途：处理可选字段**
```rust
struct Config {
    port: Option<String>,
}

fn parse_config(c: Config) -> Result<Option<u16>, ParseIntError> {
    c.port
        .map(|s| s.parse::<u16>())  // Option<Result<u16>>
        .transpose()                 // Result<Option<u16>>
}
```

---

## 六、Rust 1.70+ 新方法

```rust
let ok: Result<i32, &str> = Ok(42);

// inspect — 查看但不消耗（调试用）
ok.inspect(|x| println!("值是: {}", x))
  .map(|x| x * 2);

// inspect_err — 查看错误
err.inspect_err(|e| eprintln!("错误: {}", e));
```

---

## 实战：优雅的错误处理链

```rust
use std::fs;

fn load_user_config(user_id: u64) -> Result<Config, AppError> {
    let path = format!("/home/{}/.config/app.toml", user_id);
    
    fs::read_to_string(&path)
        .map_err(|e| AppError::Io { path: path.clone(), source: e })
        .and_then(|content| {
            toml::from_str(&content)
                .map_err(|e| AppError::Parse { path, source: e })
        })
        .inspect(|config| log::debug!("加载配置成功: {:?}", config))
        .inspect_err(|e| log::warn!("加载配置失败: {}", e))
}
```

---

## 总结表

| 方法 | 输入 | 输出 | 用途 |
|------|------|------|------|
| `unwrap` | `Result<T,E>` | `T` | 取值或 panic |
| `unwrap_or(v)` | `Result<T,E>` | `T` | 取值或默认 |
| `ok()` | `Result<T,E>` | `Option<T>` | 丢弃错误 |
| `map(f)` | `Result<T,E>` | `Result<U,E>` | 转换成功值 |
| `map_err(f)` | `Result<T,E>` | `Result<T,F>` | 转换错误 |
| `and_then(f)` | `Result<T,E>` | `Result<U,E>` | 链式 Result |
| `or_else(f)` | `Result<T,E>` | `Result<T,F>` | 错误恢复 |
| `transpose` | `Result<Option<T>,E>` | `Option<Result<T,E>>` | 类型转换 |

---

## 课后练习

写一个函数，从字符串解析出端口号：
1. 先尝试十进制解析
2. 失败了尝试十六进制（带 0x 前缀）
3. 都失败返回默认端口 8080

```rust
fn parse_port(s: &str) -> u16 {
    // 你的代码
}

assert_eq!(parse_port("3000"), 3000);
assert_eq!(parse_port("0x1F90"), 8080);
assert_eq!(parse_port("invalid"), 8080);
```

---

*下节课：std::option — Option 组合子方法大全*
