# 第 42 课：配置管理 (config crate)

> 日期：2026-02-18  
> 主题：使用 config crate 管理应用配置

---

## 为什么需要配置管理？

Web 应用通常需要：
- 数据库连接字符串
- API 密钥
- 服务端口
- 不同环境（dev/staging/prod）的不同配置

在 PHP/Laravel 中，我们有 `.env` 和 `config/*.php`。
在 Rust 中，`config` crate 提供了类似但更强大的功能。

---

## 安装依赖

```toml
[dependencies]
config = "0.14"
serde = { version = "1.0", features = ["derive"] }
dotenvy = "0.15"  # .env 文件支持
```

---

## 基础用法：定义配置结构

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct Settings {
    pub app: AppConfig,
    pub database: DatabaseConfig,
    pub server: ServerConfig,
}

#[derive(Debug, Deserialize)]
pub struct AppConfig {
    pub name: String,
    pub debug: bool,
}

#[derive(Debug, Deserialize)]
pub struct DatabaseConfig {
    pub url: String,
    pub max_connections: u32,
}

#[derive(Debug, Deserialize)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
}
```

**类型安全** - 配置直接反序列化为 Rust 结构体，编译时就能发现类型错误！

---

## 加载配置

```rust
use config::{Config, Environment, File};

impl Settings {
    pub fn new() -> Result<Self, config::ConfigError> {
        // 获取运行环境，默认 development
        let run_mode = std::env::var("RUN_MODE")
            .unwrap_or_else(|_| "development".into());

        let config = Config::builder()
            // 1. 先加载默认配置
            .add_source(File::with_name("config/default"))
            
            // 2. 根据环境加载对应配置（覆盖默认）
            .add_source(
                File::with_name(&format!("config/{}", run_mode))
                    .required(false)
            )
            
            // 3. 环境变量覆盖（前缀 APP_）
            .add_source(
                Environment::with_prefix("APP")
                    .separator("__")  // 嵌套用双下划线
            )
            
            .build()?;

        config.try_deserialize()
    }
}
```

---

## 配置文件示例

### config/default.toml

```toml
[app]
name = "my-rust-app"
debug = false

[database]
url = "postgres://localhost/myapp"
max_connections = 10

[server]
host = "0.0.0.0"
port = 8080
```

### config/development.toml

```toml
[app]
debug = true

[database]
url = "postgres://localhost/myapp_dev"
max_connections = 5
```

### config/production.toml

```toml
[database]
max_connections = 50

[server]
port = 80
```

---

## 环境变量覆盖

环境变量会覆盖文件配置：

```bash
# 覆盖 database.url
export APP_DATABASE__URL="postgres://prod-server/myapp"

# 覆盖 server.port
export APP_SERVER__PORT=3000
```

**注意**：嵌套配置用 `__`（双下划线）分隔！

---

## 集成 dotenvy

在开发环境使用 `.env` 文件：

```rust
fn main() {
    // 加载 .env 文件（如果存在）
    dotenvy::dotenv().ok();
    
    let settings = Settings::new()
        .expect("Failed to load config");
    
    println!("{:#?}", settings);
}
```

### .env 文件

```env
RUN_MODE=development
APP_DATABASE__URL=postgres://localhost/myapp_dev
APP_SERVER__PORT=3001
```

---

## 实战：全局配置单例

使用 `once_cell` 创建全局配置：

```rust
use once_cell::sync::Lazy;

pub static CONFIG: Lazy<Settings> = Lazy::new(|| {
    dotenvy::dotenv().ok();
    Settings::new().expect("Failed to load configuration")
});

// 在任何地方使用
fn somewhere() {
    println!("Port: {}", CONFIG.server.port);
    println!("Debug: {}", CONFIG.app.debug);
}
```

---

## 敏感配置处理

**不要把密钥放在代码或配置文件里！**

```rust
#[derive(Debug, Deserialize)]
pub struct Secrets {
    pub jwt_secret: String,
    pub api_key: String,
}

impl Secrets {
    pub fn from_env() -> Result<Self, config::ConfigError> {
        Config::builder()
            .add_source(Environment::with_prefix("SECRET"))
            .build()?
            .try_deserialize()
    }
}
```

```bash
export SECRET_JWT_SECRET="super-secret-key"
export SECRET_API_KEY="api-key-here"
```

---

## 配置验证

添加自定义验证：

```rust
impl Settings {
    pub fn validate(&self) -> Result<(), String> {
        if self.server.port == 0 {
            return Err("Port cannot be 0".into());
        }
        
        if self.database.max_connections < 1 {
            return Err("Need at least 1 DB connection".into());
        }
        
        Ok(())
    }
}

// 加载时验证
let settings = Settings::new()?;
settings.validate().map_err(|e| {
    config::ConfigError::Message(e)
})?;
```

---

## 配置热重载（进阶）

```rust
use config::{Config, File};
use notify::{Watcher, RecursiveMode, watcher};
use std::sync::{Arc, RwLock};

pub struct HotConfig {
    inner: Arc<RwLock<Settings>>,
}

impl HotConfig {
    pub fn get(&self) -> Settings {
        self.inner.read().unwrap().clone()
    }
    
    pub fn reload(&self) {
        if let Ok(new_config) = Settings::new() {
            *self.inner.write().unwrap() = new_config;
            println!("Config reloaded!");
        }
    }
}
```

---

## 与 Axum 集成

```rust
use axum::{Router, Extension, routing::get};
use std::sync::Arc;

#[tokio::main]
async fn main() {
    dotenvy::dotenv().ok();
    
    let config = Arc::new(Settings::new().unwrap());
    
    let app = Router::new()
        .route("/", get(handler))
        .layer(Extension(config.clone()));
    
    let addr = format!("{}:{}", 
        config.server.host, 
        config.server.port
    );
    
    println!("Server running on {}", addr);
    
    let listener = tokio::net::TcpListener::bind(&addr)
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn handler(
    Extension(config): Extension<Arc<Settings>>
) -> String {
    format!("App: {}, Debug: {}", 
        config.app.name, 
        config.app.debug
    )
}
```

---

## 对比 Laravel

| Laravel | Rust (config crate) |
|---------|---------------------|
| `config/*.php` | `config/*.toml` |
| `.env` | `.env` + dotenvy |
| `env('KEY')` | `std::env::var("KEY")` |
| `config('app.name')` | `CONFIG.app.name` |
| 运行时动态 | 编译时类型安全 ✅ |

---

## 本课要点

1. **config crate** 提供多层配置加载
2. **加载优先级**：默认 → 环境配置 → 环境变量
3. **dotenvy** 处理 `.env` 文件
4. **类型安全** - 配置即结构体
5. **敏感数据** 只用环境变量，不入代码

---

## 课后作业

创建一个配置加载模块：
1. 支持 dev/staging/prod 三个环境
2. 包含数据库、Redis、日志级别配置
3. 添加配置验证
4. 在 Axum 应用中使用

---

*下节课：Tower 中间件与 Service*
