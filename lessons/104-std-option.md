# 第 104 课：std::option — Option 组合子方法大全

## 为什么 Option 这么重要？

在 PHP/JS 里，null 是个大坑：
```php
// PHP: 运行时才知道炸了
$user = getUser();
echo $user->name;  // 💥 null pointer
```

Rust 用 `Option<T>` 逼你在编译期处理"有或没有"：
```rust
let user = get_user();  // Option<User>
// 必须显式处理 None，不然编译不过
```

---

## 一、提取值的方法

### 1. unwrap 系列

```rust
let some: Option<i32> = Some(42);
let none: Option<i32> = None;

// unwrap — 有值返回，None 直接 panic
some.unwrap()     // → 42
none.unwrap()     // → panic!

// unwrap_or — None 返回默认值
none.unwrap_or(0)           // → 0
none.unwrap_or_default()    // → 0 (i32::default())

// unwrap_or_else — None 时调用闭包（延迟计算）
none.unwrap_or_else(|| {
    eprintln!("没找到，用默认值");
    expensive_default()
})

// expect — 自定义 panic 信息
none.expect("用户必须存在")  // panic with message
```

**什么时候用 unwrap？**
- 测试代码
- 你 100% 确定是 Some
- 快速原型

---

### 2. take / replace — 原地操作

```rust
let mut slot: Option<String> = Some("hello".to_string());

// take — 拿走值，原地留 None
let taken = slot.take();  // taken = Some("hello")
                          // slot = None

// replace — 换个新值，返回旧值
let mut slot = Some(1);
let old = slot.replace(2);  // old = Some(1), slot = Some(2)
```

**实战：一次性资源**
```rust
struct Connection {
    inner: Option<TcpStream>,
}

impl Connection {
    fn close(&mut self) -> Option<TcpStream> {
        self.inner.take()  // 拿走后就没了
    }
}
```

---

## 二、检查状态

```rust
let some: Option<i32> = Some(42);
let none: Option<i32> = None;

// 基础检查
some.is_some()  // → true
some.is_none()  // → false

// is_some_and (Rust 1.70+) — 检查 + 条件
some.is_some_and(|x| x > 40)   // → true
some.is_some_and(|x| x > 50)   // → false
none.is_some_and(|x| x > 0)    // → false (None 直接返回 false)
```

**用途：条件检查一行搞定**
```rust
// 老写法
if let Some(user) = get_user() {
    if user.is_admin() {
        // ...
    }
}

// 新写法
if get_user().is_some_and(|u| u.is_admin()) {
    // ...
}
```

---

## 三、转换值的方法

### 1. map — 转换内部值

```rust
let some: Option<String> = Some("42".to_string());

// map — Some 时转换，None 不变
some.map(|s| s.len())  // → Some(2)
none.map(|s| s.len())  // → None

// 链式转换
Some("  hello  ")
    .map(|s| s.trim())
    .map(|s| s.to_uppercase())
// → Some("HELLO")
```

### 2. map_or / map_or_else — 带默认值的转换

```rust
let some: Option<i32> = Some(42);
let none: Option<i32> = None;

// map_or — Some 转换，None 返回默认
some.map_or(0, |x| x * 2)   // → 84
none.map_or(0, |x| x * 2)   // → 0

// map_or_else — 两边都是闭包
none.map_or_else(
    || expensive_default(),  // None 分支
    |x| x * 2                // Some 分支
)
```

**对比 JS：**
```javascript
// JS 可选链 + 空值合并
user?.profile?.avatar ?? defaultAvatar

// Rust
user.as_ref()
    .and_then(|u| u.profile.as_ref())
    .map(|p| &p.avatar)
    .unwrap_or(&default_avatar)
```

---

## 四、链式操作

### 1. and_then — 链接返回 Option 的操作

```rust
fn parse_int(s: &str) -> Option<i32> {
    s.parse().ok()
}

fn double_if_positive(n: i32) -> Option<i32> {
    if n > 0 { Some(n * 2) } else { None }
}

// 链式调用
Some("42")
    .and_then(parse_int)
    .and_then(double_if_positive)
// → Some(84)

Some("-5")
    .and_then(parse_int)
    .and_then(double_if_positive)
// → None (负数被过滤掉了)
```

**map vs and_then 的区别：**
```rust
// map: 闭包返回 T
option.map(|x| x * 2)  // Option<i32> → Option<i32>

// and_then: 闭包返回 Option<T>
option.and_then(|x| validate(x))  // Option<i32> → Option<i32>
                                  // 不会变成 Option<Option<i32>>
```

### 2. or / or_else — 没有时的备选

```rust
let none: Option<i32> = None;

// or — None 时用备选值
none.or(Some(42))  // → Some(42)

// or_else — None 时调用闭包
none.or_else(|| {
    println!("没找到，查数据库...");
    db.find_default()
})
```

**实战：配置优先级**
```rust
fn get_port() -> u16 {
    env::var("PORT").ok()
        .and_then(|s| s.parse().ok())    // 环境变量
        .or_else(|| config.port)          // 配置文件
        .or(Some(8080))                   // 默认值
        .unwrap()
}
```

### 3. filter — 条件过滤

```rust
Some(42).filter(|x| x > 40)   // → Some(42)
Some(42).filter(|x| x > 50)   // → None
None.filter(|_| true)         // → None
```

**实战：验证输入**
```rust
fn parse_positive_int(s: &str) -> Option<i32> {
    s.parse::<i32>().ok()
        .filter(|&n| n > 0)
}
```

---

## 五、和 Result 互转

### 1. ok_or / ok_or_else — Option → Result

```rust
let some: Option<i32> = Some(42);
let none: Option<i32> = None;

// ok_or — None 变成指定错误
some.ok_or("not found")?  // → 42
none.ok_or("not found")?  // → Err("not found")

// ok_or_else — 延迟构造错误
none.ok_or_else(|| format!("user {} not found", user_id))
```

**这是最常用的模式！**
```rust
fn get_user(id: u64) -> Result<User, AppError> {
    users.get(&id)
        .ok_or(AppError::NotFound { id })?
        .clone()
        .pipe(Ok)
}
```

### 2. transpose — Option<Result> ↔ Result<Option>

```rust
let x: Option<Result<i32, &str>> = Some(Ok(42));
let y: Result<Option<i32>, &str> = x.transpose();
// y = Ok(Some(42))

let x: Option<Result<i32, &str>> = None;
let y: Result<Option<i32>, &str> = x.transpose();
// y = Ok(None)
```

---

## 六、引用相关

### 1. as_ref / as_mut

```rust
let opt: Option<String> = Some("hello".to_string());

// as_ref: Option<T> → Option<&T>
opt.as_ref().map(|s| s.len())  // 不消耗 opt

// as_mut: Option<T> → Option<&mut T>
let mut opt = Some(vec![1, 2, 3]);
opt.as_mut().map(|v| v.push(4));
```

### 2. as_deref / as_deref_mut

```rust
let opt: Option<String> = Some("hello".to_string());

// as_deref: Option<String> → Option<&str>
opt.as_deref()  // → Some("hello") 类型是 Option<&str>

// 常用于 String/Vec 等智能指针类型
let opt: Option<Vec<i32>> = Some(vec![1, 2, 3]);
opt.as_deref()  // → Some(&[1, 2, 3]) 类型是 Option<&[i32]>
```

---

## 七、迭代相关

```rust
let some: Option<i32> = Some(42);
let none: Option<i32> = None;

// iter — 转成最多一个元素的迭代器
for x in some.iter() {
    println!("{}", x);  // 打印 42
}

// into_iter — 消耗 Option
for x in some {
    println!("{}", x);  // 打印 42
}

// 配合 flatten 过滤 None
let opts = vec![Some(1), None, Some(3)];
let values: Vec<_> = opts.into_iter().flatten().collect();
// → [1, 3]
```

---

## 八、实战组合技

### 1. 深度嵌套取值
```rust
struct Company {
    ceo: Option<Person>,
}
struct Person {
    contact: Option<Contact>,
}
struct Contact {
    email: Option<String>,
}

fn get_ceo_email(company: &Company) -> Option<&str> {
    company.ceo.as_ref()
        .and_then(|p| p.contact.as_ref())
        .and_then(|c| c.email.as_deref())
}
```

### 2. 配置解析
```rust
fn load_config() -> Config {
    Config {
        port: env::var("PORT").ok()
            .and_then(|s| s.parse().ok())
            .unwrap_or(8080),
        
        host: env::var("HOST").ok()
            .filter(|s| !s.is_empty())
            .unwrap_or_else(|| "127.0.0.1".to_string()),
        
        debug: env::var("DEBUG").ok()
            .map(|s| s == "1" || s == "true")
            .unwrap_or(false),
    }
}
```

### 3. 数据库查询封装
```rust
impl UserRepo {
    fn find_by_email(&self, email: &str) -> Result<Option<User>, DbError> {
        self.query("SELECT * FROM users WHERE email = ?", &[email])
            .map(|rows| rows.first().cloned())
    }
    
    fn must_find(&self, email: &str) -> Result<User, AppError> {
        self.find_by_email(email)?
            .ok_or_else(|| AppError::UserNotFound { email: email.to_string() })
    }
}
```

---

## 九、Rust 1.70+ 新方法

```rust
let some: Option<i32> = Some(42);

// inspect — 查看但不消耗（调试用）
some.inspect(|x| println!("值是: {}", x))
    .map(|x| x * 2);
// 打印 "值是: 42"，然后返回 Some(84)
```

---

## 总结对照表

| 方法 | 输入 | 输出 | 用途 |
|------|------|------|------|
| `unwrap` | `Option<T>` | `T` | 取值或 panic |
| `unwrap_or(v)` | `Option<T>` | `T` | 取值或默认 |
| `unwrap_or_else(f)` | `Option<T>` | `T` | 取值或延迟计算默认 |
| `take` | `&mut Option<T>` | `Option<T>` | 拿走值 |
| `map(f)` | `Option<T>` | `Option<U>` | 转换值 |
| `map_or(v, f)` | `Option<T>` | `U` | 转换或默认 |
| `and_then(f)` | `Option<T>` | `Option<U>` | 链式 Option |
| `or(v)` | `Option<T>` | `Option<T>` | 备选值 |
| `or_else(f)` | `Option<T>` | `Option<T>` | 延迟备选 |
| `filter(p)` | `Option<T>` | `Option<T>` | 条件过滤 |
| `ok_or(e)` | `Option<T>` | `Result<T,E>` | 转 Result |
| `transpose` | `Option<Result>` | `Result<Option>` | 类型转换 |
| `as_ref` | `&Option<T>` | `Option<&T>` | 借用内部值 |
| `as_deref` | `&Option<T>` | `Option<&T::Target>` | 智能指针解引用 |

---

## 课后练习

实现一个函数，从环境变量读取数据库 URL：
1. 先尝试 `DATABASE_URL`
2. 没有就拼接 `DB_HOST`、`DB_PORT`、`DB_NAME`
3. 都没有返回默认值

```rust
fn get_db_url() -> String {
    // 你的代码
}

// 测试
// DATABASE_URL=mysql://root@localhost/test → "mysql://root@localhost/test"
// DB_HOST=127.0.0.1, DB_PORT=3306, DB_NAME=app → "mysql://127.0.0.1:3306/app"
// 都没有 → "mysql://localhost:3306/default"
```

<details>
<summary>参考答案</summary>

```rust
use std::env;

fn get_db_url() -> String {
    env::var("DATABASE_URL").ok()
        .or_else(|| {
            let host = env::var("DB_HOST").ok()?;
            let port = env::var("DB_PORT").ok().unwrap_or_else(|| "3306".to_string());
            let name = env::var("DB_NAME").ok()?;
            Some(format!("mysql://{}:{}/{}", host, port, name))
        })
        .unwrap_or_else(|| "mysql://localhost:3306/default".to_string())
}
```

</details>

---

*下节课：std::slice — 切片方法大全*
