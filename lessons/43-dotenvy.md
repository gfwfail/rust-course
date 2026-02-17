# 第 43 课：环境变量管理 (dotenvy)

> 授课时间：2026-02-18  
> 关键词：dotenvy, 环境变量, .env, 配置安全

---

## 📦 为什么要用 dotenvy？

```
场景：你的程序需要读取敏感信息（数据库密码、API Key）

❌ 硬编码在代码里 → 泄露风险
❌ 写在 config 文件提交到 Git → 一样危险
✅ 用 .env 文件 + .gitignore → 安全！
```

**dotenvy vs dotenv：**
- `dotenv` 是老库，已停止维护
- `dotenvy` 是社区 fork，积极维护中
- 用 `dotenvy`，别用 `dotenv`！

---

## 🚀 快速开始

**Cargo.toml:**
```toml
[dependencies]
dotenvy = "0.15"
```

**项目根目录创建 .env:**
```env
DATABASE_URL=postgres://user:pass@localhost/mydb
API_KEY=sk-1234567890abcdef
DEBUG=true
PORT=8080
```

**main.rs:**
```rust
use std::env;

fn main() {
    // 加载 .env 文件到环境变量
    dotenvy::dotenv().ok();
    
    // 现在可以用 std::env 读取了
    let db_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    let port: u16 = env::var("PORT")
        .unwrap_or_else(|_| "3000".to_string())
        .parse()
        .expect("PORT must be a number");
    
    println!("Connecting to: {}", db_url);
    println!("Server port: {}", port);
}
```

---

## 💡 关键点解析

### 1. `.ok()` 的妙用

```rust
// 方式一：忽略错误（文件不存在也没事）
dotenvy::dotenv().ok();

// 方式二：必须有 .env 文件
dotenvy::dotenv().expect(".env file not found");

// 方式三：检查是否加载成功
if dotenvy::dotenv().is_ok() {
    println!("Loaded .env file");
} else {
    println!("No .env file, using system env");
}
```

**推荐用 `.ok()`** —— 生产环境通常通过系统环境变量注入，不需要 .env 文件。

### 2. 不会覆盖已有环境变量

```rust
// 如果系统已设置 DATABASE_URL，.env 里的值不会覆盖它
// 这是正确的行为！生产环境系统变量优先
```

这点很重要：**系统环境变量 > .env 文件**

### 3. 指定其他文件

```rust
// 加载特定文件
dotenvy::from_filename(".env.local").ok();

// 加载指定路径
dotenvy::from_path("/etc/myapp/.env").ok();
```

---

## 🔧 实战：类型安全的配置

直接用 `env::var` 到处写很丑，来封装一下：

```rust
use std::env;

pub struct Config {
    pub database_url: String,
    pub api_key: String,
    pub debug: bool,
    pub port: u16,
}

impl Config {
    pub fn from_env() -> Result<Self, env::VarError> {
        Ok(Config {
            database_url: env::var("DATABASE_URL")?,
            api_key: env::var("API_KEY")?,
            debug: env::var("DEBUG")
                .map(|v| v == "true")
                .unwrap_or(false),
            port: env::var("PORT")
                .unwrap_or_else(|_| "3000".to_string())
                .parse()
                .unwrap_or(3000),
        })
    }
}

fn main() {
    dotenvy::dotenv().ok();
    
    let config = Config::from_env()
        .expect("Failed to load config");
    
    println!("Debug mode: {}", config.debug);
}
```

---

## 🤝 与 config crate 配合

上节课的 config crate 可以直接读环境变量：

```rust
use config::{Config, Environment};

fn main() {
    dotenvy::dotenv().ok();  // 先加载 .env
    
    let settings = Config::builder()
        .add_source(Environment::default())
        .build()
        .unwrap();
    
    let db: String = settings.get("database_url").unwrap();
}
```

**最佳实践：**
- 开发环境：`.env` 文件
- 生产环境：系统环境变量（Docker、K8s、Laravel Cloud 等）

---

## ⚠️ .gitignore 必须加！

```gitignore
# .gitignore
.env
.env.local
.env.*.local
```

可以提交一个示例文件：
```bash
# .env.example（提交到 Git）
DATABASE_URL=postgres://user:pass@localhost/mydb
API_KEY=your-api-key-here
DEBUG=false
PORT=3000
```

---

## 📝 课后小结

| 概念 | 说明 |
|------|------|
| `dotenvy::dotenv()` | 加载 .env 到环境变量 |
| `.ok()` | 忽略加载失败（推荐） |
| `env::var()` | 读取环境变量 |
| 优先级 | 系统环境变量 > .env |

**Laravel 对比：**
- Laravel: `env('DATABASE_URL')` 
- Rust: `env::var("DATABASE_URL")`
- 都是先加载 .env，系统变量优先

---

## 🔗 相关资源

- [dotenvy crate](https://crates.io/crates/dotenvy)
- [12-Factor App: Config](https://12factor.net/config)

---

*下节课预告：项目结构最佳实践*
