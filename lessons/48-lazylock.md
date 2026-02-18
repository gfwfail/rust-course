# 第 48 课：全局状态与延迟初始化 (LazyLock)

> 日期：2026-02-18
> 主题：`std::sync::LazyLock` / `OnceLock` / `once_cell`

---

## 📌 问题背景

Web 开发经常需要全局配置、连接池、缓存客户端。

PHP/Laravel 很简单：
```php
$redis = Redis::connection();
config('app.name');
```

但 Rust 静态变量必须编译时初始化：
```rust
// ❌ 编译错误！
static CONFIG: Config = load_config();  // 不能在编译时执行函数
```

---

## 🧠 核心概念

### 问题：Rust 静态变量的限制

```rust
// ✅ 编译时常量
static MAX_SIZE: usize = 1024;

// ❌ 需要运行时计算
static CONFIG: Config = Config::load();  // Error!
```

Rust 要求 `static` 变量必须是编译时常量（const），这是为了内存安全。

### 解决方案演进

**2021 年前**：使用 `lazy_static!` 宏
```rust
use lazy_static::lazy_static;

lazy_static! {
    static ref CONFIG: Config = Config::load();
}
```

**2021-2024**：使用 `once_cell`（更现代）
```rust
use once_cell::sync::Lazy;

static CONFIG: Lazy<Config> = Lazy::new(|| Config::load());
```

**Rust 1.80+（2024年7月）**：标准库内置！
```rust
use std::sync::LazyLock;

static CONFIG: LazyLock<Config> = LazyLock::new(|| Config::load());
```

---

## 💻 实战代码

### 1️⃣ 全局配置

```rust
use std::sync::LazyLock;

#[derive(Debug)]
struct AppConfig {
    database_url: String,
    api_key: String,
    max_connections: usize,
}

impl AppConfig {
    fn load() -> Self {
        println!("⚙️ 加载配置（只执行一次）");
        Self {
            database_url: std::env::var("DATABASE_URL")
                .unwrap_or_else(|_| "postgres://localhost/app".into()),
            api_key: std::env::var("API_KEY")
                .unwrap_or_else(|_| "dev-key".into()),
            max_connections: 10,
        }
    }
}

// 全局配置 - 首次访问时初始化，之后复用
static CONFIG: LazyLock<AppConfig> = LazyLock::new(AppConfig::load);

fn main() {
    println!("程序启动...");
    
    // 首次访问 - 触发初始化
    println!("DB: {}", CONFIG.database_url);
    
    // 再次访问 - 直接复用，不会重新初始化
    println!("Key: {}", CONFIG.api_key);
    println!("Max: {}", CONFIG.max_connections);
}
```

输出：
```
程序启动...
⚙️ 加载配置（只执行一次）
DB: postgres://localhost/app
Key: dev-key
Max: 10
```

---

### 2️⃣ 正则表达式预编译

正则编译很耗时，全局预编译是最佳实践：

```rust
use std::sync::LazyLock;
use regex::Regex;

// 预编译正则 - 程序生命周期内只编译一次
static EMAIL_RE: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r"^[\w\.-]+@[\w\.-]+\.\w+$").unwrap()
});

static PHONE_RE: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r"^\d{11}$").unwrap()
});

fn validate_email(email: &str) -> bool {
    EMAIL_RE.is_match(email)
}

fn validate_phone(phone: &str) -> bool {
    PHONE_RE.is_match(phone)
}

fn main() {
    let emails = ["test@example.com", "invalid-email", "a@b.c"];
    
    for email in emails {
        println!("{}: {}", email, validate_email(email));
    }
}
```

---

### 3️⃣ HTTP 客户端单例

```rust
use std::sync::LazyLock;
use std::time::Duration;

// 全局 HTTP 客户端（复用连接池）
static HTTP_CLIENT: LazyLock<reqwest::Client> = LazyLock::new(|| {
    reqwest::Client::builder()
        .timeout(Duration::from_secs(30))
        .pool_max_idle_per_host(10)
        .build()
        .expect("Failed to create HTTP client")
});

async fn fetch_data(url: &str) -> reqwest::Result<String> {
    // 使用全局客户端，复用连接
    HTTP_CLIENT.get(url).send().await?.text().await
}
```

---

## 🆚 LazyLock vs OnceLock

标准库提供两个类型：

| 类型 | 用途 | 初始化 |
|------|------|--------|
| `LazyLock<T>` | 静态变量 | 声明时提供闭包 |
| `OnceLock<T>` | 灵活控制 | 运行时决定值 |

```rust
use std::sync::{LazyLock, OnceLock};

// LazyLock - 闭包固定
static A: LazyLock<String> = LazyLock::new(|| "hello".into());

// OnceLock - 运行时设置
static B: OnceLock<String> = OnceLock::new();

fn main() {
    // LazyLock 自动初始化
    println!("{}", *A);
    
    // OnceLock 需要手动设置
    B.get_or_init(|| "world".into());
    println!("{}", B.get().unwrap());
    
    // 第二次设置无效（保持第一次的值）
    B.get_or_init(|| "ignored".into());
    println!("{}", B.get().unwrap());  // 仍然是 "world"
}
```

---

## ⚠️ 注意事项

### 1. 线程安全
`LazyLock` 和 `OnceLock` 都是线程安全的（`Sync`）。
多线程首次访问时，只有一个线程执行初始化，其他线程等待。

### 2. 非线程安全版本
如果确定单线程，可以用 `std::cell::LazyCell`（无锁，更快）：
```rust
use std::cell::LazyCell;

// ⚠️ 只能用于单线程！
thread_local! {
    static LOCAL_CONFIG: LazyCell<Config> = LazyCell::new(|| Config::load());
}
```

### 3. 初始化 panic
如果初始化闭包 panic，之后每次访问都会 panic！

```rust
static BAD: LazyLock<i32> = LazyLock::new(|| panic!("boom"));

fn main() {
    // 第一次：panic
    // let _ = *BAD;
    
    // 后续访问：继续 panic（已中毒）
}
```

### 4. 全局可变状态
需要配合 `Mutex`：

```rust
use std::sync::{LazyLock, Mutex};

// 全局可变状态
static COUNTER: LazyLock<Mutex<u64>> = LazyLock::new(|| Mutex::new(0));

fn increment() -> u64 {
    let mut guard = COUNTER.lock().unwrap();
    *guard += 1;
    *guard
}
```

---

## 📊 对比总结

| 方案 | 优点 | 缺点 | 推荐场景 |
|------|------|------|----------|
| `LazyLock` (std) | 标准库内置，无依赖 | Rust 1.80+ | 新项目首选 |
| `once_cell::Lazy` | 兼容旧版本 | 额外依赖 | 需要支持旧 Rust |
| `lazy_static!` | 历史悠久 | 宏语法丑 | 遗留代码 |

---

## 🎯 最佳实践

1. **配置/连接池** → 用 `LazyLock` 全局单例
2. **预编译正则** → 用 `LazyLock<Regex>`
3. **可变全局状态** → 用 `LazyLock<Mutex<T>>`
4. **运行时决定值** → 用 `OnceLock`

---

## 📝 小结

- Rust 静态变量必须编译时常量 → 需要延迟初始化
- `LazyLock`：声明时提供闭包，首次访问时初始化
- `OnceLock`：运行时灵活设置值
- Rust 1.80+ 标准库内置，不再需要 `once_cell`
- 常见用途：全局配置、HTTP 客户端、正则预编译、数据库连接池

---

*下节课预告：rand 随机数生成*
