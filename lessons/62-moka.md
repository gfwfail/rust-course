# 第 62 课：内存缓存 moka

## 📌 为什么需要内存缓存？

在 Web 开发中，有些数据：
- 查询频繁但变化不大（用户配置、商品信息）
- 计算代价高（聚合统计、复杂查询）
- 外部 API 有限流（第三方服务调用）

每次都查数据库或调 API？太慢！内存缓存是解决方案。

---

## 🎯 moka 是什么？

**moka** 是 Rust 生态最强大的内存缓存库，灵感来自 Java 的 Caffeine。

特点：
- ⚡ 高性能并发访问（lock-free 数据结构）
- ⏰ 支持 TTL（过期时间）和 TTI（空闲过期）
- 📊 容量限制 + 智能淘汰（TinyLFU 算法）
- 🔄 async 支持
- 📈 可选统计功能

---

## 📦 添加依赖

```toml
[dependencies]
moka = { version = "0.12", features = ["future"] }  # async 支持
tokio = { version = "1", features = ["full"] }
```

---

## 🔰 基础用法：同步缓存

```rust
use moka::sync::Cache;
use std::time::Duration;

fn main() {
    // 创建缓存：最多 100 个条目，5 分钟过期
    let cache: Cache<String, String> = Cache::builder()
        .max_capacity(100)
        .time_to_live(Duration::from_secs(300))
        .build();

    // 插入
    cache.insert("user:1".to_string(), "Alice".to_string());

    // 获取
    if let Some(name) = cache.get(&"user:1".to_string()) {
        println!("找到用户: {}", name);
    }

    // 获取或计算（最常用！）
    let value = cache.get_with("user:2".to_string(), || {
        println!("缓存未命中，从数据库查询...");
        "Bob".to_string()
    });
    println!("用户: {}", value);

    // 删除
    cache.invalidate(&"user:1".to_string());

    // 清空所有
    cache.invalidate_all();
}
```

---

## ⚡ 异步缓存（Web 开发必用）

```rust
use moka::future::Cache;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let cache: Cache<i64, User> = Cache::builder()
        .max_capacity(1000)
        .time_to_live(Duration::from_secs(600))
        .build();

    // 异步获取或计算
    let user = cache.get_with(42, async {
        // 模拟数据库查询
        fetch_user_from_db(42).await
    }).await;

    println!("用户: {:?}", user);
}

#[derive(Clone, Debug)]
struct User {
    id: i64,
    name: String,
}

async fn fetch_user_from_db(id: i64) -> User {
    // 实际项目中这里是 SQLx 查询
    tokio::time::sleep(Duration::from_millis(100)).await;
    User { id, name: format!("User{}", id) }
}
```

---

## 🏗️ 实战：Axum + moka 缓存层

```rust
use axum::{
    extract::{Path, State},
    routing::get,
    Router, Json,
};
use moka::future::Cache;
use std::{sync::Arc, time::Duration};
use serde::{Deserialize, Serialize};

#[derive(Clone, Serialize, Deserialize)]
struct Product {
    id: i64,
    name: String,
    price: f64,
}

// 应用状态：包含缓存
#[derive(Clone)]
struct AppState {
    product_cache: Cache<i64, Product>,
    // db_pool: PgPool,  // 实际项目中
}

#[tokio::main]
async fn main() {
    let state = AppState {
        product_cache: Cache::builder()
            .max_capacity(10_000)
            .time_to_live(Duration::from_secs(300))  // 5 分钟过期
            .time_to_idle(Duration::from_secs(60))   // 1 分钟无访问也过期
            .build(),
    };

    let app = Router::new()
        .route("/products/:id", get(get_product))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn get_product(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Json<Product> {
    // 缓存命中直接返回，未命中则查库并缓存
    let product = state.product_cache.get_with(id, async {
        // 实际项目：sqlx::query_as!(...)
        fetch_product_from_db(id).await
    }).await;

    Json(product)
}

async fn fetch_product_from_db(id: i64) -> Product {
    println!("📀 查询数据库: product_id={}", id);
    Product {
        id,
        name: format!("商品{}", id),
        price: 99.99,
    }
}
```

---

## 🔧 进阶配置

```rust
use moka::future::Cache;
use moka::notification::RemovalCause;
use std::time::Duration;

let cache: Cache<String, String> = Cache::builder()
    // 容量限制
    .max_capacity(10_000)
    
    // 按权重计算容量（大对象占更多配额）
    .weigher(|_key, value: &String| value.len() as u32)
    
    // 绝对过期时间（插入后 N 秒过期）
    .time_to_live(Duration::from_secs(3600))
    
    // 空闲过期时间（最后访问后 N 秒过期）
    .time_to_idle(Duration::from_secs(300))
    
    // 淘汰回调（可用于记录日志、同步到持久化层）
    .eviction_listener(|key, value, cause| {
        match cause {
            RemovalCause::Expired => println!("过期: {}", key),
            RemovalCause::Size => println!("容量淘汰: {}", key),
            RemovalCause::Explicit => println!("手动删除: {}", key),
            _ => {}
        }
    })
    
    .build();
```

---

## 📊 统计功能

```rust
// 启用统计
let cache: Cache<String, String> = Cache::builder()
    .max_capacity(1000)
    .record_statistics()
    .build();

// 进行一些操作...
cache.insert("a".into(), "1".into());
cache.get(&"a".into());
cache.get(&"b".into());  // miss

// 打印统计
if let Some(stats) = cache.stats() {
    println!("命中次数: {}", stats.hits());
    println!("未命中: {}", stats.misses());
    println!("命中率: {:.2}%", stats.hit_ratio() * 100.0);
}
```

---

## 💡 常见模式

### 1. 缓存击穿防护（get_with 自带）

```rust
// get_with 保证同一个 key 只有一个请求会执行计算
// 其他请求等待结果，避免缓存击穿
let value = cache.get_with(key, async {
    expensive_computation().await
}).await;
```

### 2. 手动刷新

```rust
// 强制刷新某个 key
async fn refresh_product(cache: &Cache<i64, Product>, id: i64) {
    let fresh = fetch_product_from_db(id).await;
    cache.insert(id, fresh);
}
```

### 3. 多级缓存键

```rust
// 用 tuple 或自定义类型作为 key
let cache: Cache<(String, i64), Data> = Cache::builder()
    .max_capacity(10000)
    .build();

cache.insert(("user_orders".into(), 42), orders);
```

---

## ⚠️ 注意事项

1. **缓存值必须 Clone** - moka 返回的是克隆的值
2. **避免缓存大对象** - 考虑用 `Arc<T>` 包装
3. **设置合理 TTL** - 太长数据过期，太短命中率低
4. **监控命中率** - 低于 80% 就要调整策略

```rust
// 用 Arc 避免大对象克隆
let cache: Cache<i64, Arc<LargeData>> = Cache::builder()
    .max_capacity(100)
    .build();
```

---

## 🆚 对比其他方案

| 方案 | 场景 | 优缺点 |
|------|------|--------|
| **moka** | 单机内存缓存 | 最快，但重启丢失 |
| **Redis** | 分布式缓存 | 可持久化，但有网络开销 |
| **数据库** | 持久化存储 | 最慢，但数据可靠 |

最佳实践：**moka + Redis 两级缓存**
- L1: moka（毫秒级，本进程）
- L2: Redis（毫秒~几十毫秒，跨进程）

---

## 📝 本课小结

1. **moka** 是 Rust 最好的内存缓存库
2. 用 `get_with` 实现"缓存未命中则计算"
3. 合理设置 `max_capacity`、`time_to_live`、`time_to_idle`
4. 缓存值用 `Arc` 包装避免大对象克隆
5. 生产环境开启统计，监控命中率

---

*课程日期：2026-02-20*
