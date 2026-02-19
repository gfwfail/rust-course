# 第 54 课：Redis 集成 (redis crate)

> 日期：2026-02-19  
> 主题：Redis 集成与缓存操作

---

## 📌 为什么要学 Redis？

Web 开发中，Redis 无处不在：
- **缓存** — 减少数据库压力
- **Session 存储** — 分布式会话管理
- **消息队列** — Pub/Sub 实时通信
- **限流** — Rate limiting
- **分布式锁** — 防止并发冲突

PHP 里用 `Redis::get()`，Rust 里用 `redis` crate。

---

## 🔧 添加依赖

```toml
[dependencies]
redis = { version = "0.25", features = ["tokio-comp", "connection-manager"] }
tokio = { version = "1", features = ["full"] }
```

- `tokio-comp` — 异步支持
- `connection-manager` — 连接池管理

---

## 🚀 基础连接

```rust
use redis::AsyncCommands;

#[tokio::main]
async fn main() -> redis::RedisResult<()> {
    // 连接 Redis
    let client = redis::Client::open("redis://127.0.0.1:6379/")?;
    let mut con = client.get_multiplexed_async_connection().await?;
    
    // SET key value
    con.set("my_key", "Hello Rust!").await?;
    
    // GET key
    let value: String = con.get("my_key").await?;
    println!("Value: {}", value); // Hello Rust!
    
    Ok(())
}
```

### 连接字符串格式

```
redis://[[username:]password@]host[:port][/database]
redis://default:mypassword@localhost:6379/0
```

---

## 📦 常用操作

### 1️⃣ 字符串操作

```rust
use redis::AsyncCommands;

async fn string_ops(con: &mut redis::aio::MultiplexedConnection) 
    -> redis::RedisResult<()> 
{
    // SET with expiration (秒)
    con.set_ex("session:abc123", "user_data", 3600).await?;
    
    // SETEX 等效写法
    redis::cmd("SETEX")
        .arg("temp_key")
        .arg(60)          // 60 秒过期
        .arg("temp_value")
        .exec_async(con)
        .await?;
    
    // SETNX - 不存在才设置
    let was_set: bool = con.set_nx("unique_key", "value").await?;
    
    // INCR / DECR
    con.set("counter", 0i64).await?;
    let new_val: i64 = con.incr("counter", 1).await?;
    println!("Counter: {}", new_val); // 1
    
    // MSET / MGET - 批量操作
    con.mset(&[("k1", "v1"), ("k2", "v2"), ("k3", "v3")]).await?;
    let values: Vec<String> = con.mget(&["k1", "k2", "k3"]).await?;
    
    Ok(())
}
```

### 2️⃣ Hash 操作

```rust
use redis::AsyncCommands;
use std::collections::HashMap;

async fn hash_ops(con: &mut redis::aio::MultiplexedConnection) 
    -> redis::RedisResult<()> 
{
    // HSET - 设置单个字段
    con.hset("user:1001", "name", "Alice").await?;
    con.hset("user:1001", "email", "alice@example.com").await?;
    
    // HSET 多个字段
    con.hset_multiple("user:1002", &[
        ("name", "Bob"),
        ("email", "bob@example.com"),
        ("role", "admin"),
    ]).await?;
    
    // HGET - 获取单个字段
    let name: String = con.hget("user:1001", "name").await?;
    
    // HGETALL - 获取所有字段
    let user: HashMap<String, String> = con.hgetall("user:1002").await?;
    for (k, v) in &user {
        println!("{}: {}", k, v);
    }
    
    // HDEL - 删除字段
    con.hdel("user:1002", "role").await?;
    
    // HEXISTS - 字段是否存在
    let exists: bool = con.hexists("user:1001", "name").await?;
    
    Ok(())
}
```

### 3️⃣ List 操作（队列）

```rust
async fn list_ops(con: &mut redis::aio::MultiplexedConnection) 
    -> redis::RedisResult<()> 
{
    // LPUSH - 左侧插入（队列头）
    con.lpush("queue:tasks", "task3").await?;
    con.lpush("queue:tasks", "task2").await?;
    con.lpush("queue:tasks", "task1").await?;
    
    // RPUSH - 右侧插入（队列尾）
    con.rpush("queue:tasks", "task4").await?;
    
    // RPOP - 右侧弹出（FIFO 队列消费）
    let task: Option<String> = con.rpop("queue:tasks", None).await?;
    
    // BRPOP - 阻塞式弹出（等待新任务）
    // 超时 0 = 永久等待
    let result: Option<(String, String)> = redis::cmd("BRPOP")
        .arg("queue:tasks")
        .arg(5)  // 5 秒超时
        .query_async(con)
        .await?;
    
    // LRANGE - 获取范围
    let all: Vec<String> = con.lrange("queue:tasks", 0, -1).await?;
    
    // LLEN - 队列长度
    let len: i64 = con.llen("queue:tasks").await?;
    
    Ok(())
}
```

---

## 🔐 实战：分布式锁

```rust
use redis::AsyncCommands;
use std::time::Duration;

/// 获取分布式锁
async fn acquire_lock(
    con: &mut redis::aio::MultiplexedConnection,
    lock_key: &str,
    lock_value: &str,
    ttl_seconds: u64,
) -> redis::RedisResult<bool> {
    // SET key value NX EX seconds
    let result: Option<String> = redis::cmd("SET")
        .arg(lock_key)
        .arg(lock_value)
        .arg("NX")              // 不存在才设置
        .arg("EX")
        .arg(ttl_seconds)       // 过期时间
        .query_async(con)
        .await?;
    
    Ok(result.is_some())
}

/// 释放分布式锁（带 Lua 脚本保证原子性）
async fn release_lock(
    con: &mut redis::aio::MultiplexedConnection,
    lock_key: &str,
    lock_value: &str,
) -> redis::RedisResult<bool> {
    // Lua 脚本：只有 value 匹配才删除
    let script = r#"
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    "#;
    
    let result: i32 = redis::Script::new(script)
        .key(lock_key)
        .arg(lock_value)
        .invoke_async(con)
        .await?;
    
    Ok(result == 1)
}

// 使用示例
async fn do_critical_work(
    con: &mut redis::aio::MultiplexedConnection,
) -> redis::RedisResult<()> {
    let lock_key = "lock:order:12345";
    let lock_value = uuid::Uuid::new_v4().to_string();
    
    // 尝试获取锁
    if acquire_lock(con, lock_key, &lock_value, 30).await? {
        println!("获取锁成功，执行关键操作...");
        
        // 执行业务逻辑
        tokio::time::sleep(Duration::from_secs(2)).await;
        
        // 释放锁
        release_lock(con, lock_key, &lock_value).await?;
        println!("锁已释放");
    } else {
        println!("获取锁失败，有其他进程在处理");
    }
    
    Ok(())
}
```

---

## 📡 Pub/Sub 消息订阅

```rust
use redis::AsyncCommands;

// 发布者
async fn publisher(client: &redis::Client) -> redis::RedisResult<()> {
    let mut con = client.get_multiplexed_async_connection().await?;
    
    // 发布消息
    con.publish("channel:notifications", "新订单来了！").await?;
    con.publish("channel:notifications", "用户注册成功").await?;
    
    Ok(())
}

// 订阅者
async fn subscriber(client: &redis::Client) -> redis::RedisResult<()> {
    let mut pubsub = client.get_async_pubsub().await?;
    
    // 订阅频道
    pubsub.subscribe("channel:notifications").await?;
    
    // 监听消息
    let mut stream = pubsub.on_message();
    while let Some(msg) = stream.next().await {
        let channel: String = msg.get_channel()?;
        let payload: String = msg.get_payload()?;
        println!("[{}] {}", channel, payload);
    }
    
    Ok(())
}
```

---

## 🏗️ Axum 集成

```rust
use axum::{
    extract::State,
    routing::get,
    Router,
    Json,
};
use redis::aio::MultiplexedConnection;
use redis::AsyncCommands;
use std::sync::Arc;
use tokio::sync::Mutex;

// 应用状态
struct AppState {
    redis: Mutex<MultiplexedConnection>,
}

async fn get_value(
    State(state): State<Arc<AppState>>,
) -> Json<String> {
    let mut con = state.redis.lock().await;
    let value: String = con.get("my_key").await.unwrap_or_default();
    Json(value)
}

#[tokio::main]
async fn main() {
    let client = redis::Client::open("redis://127.0.0.1/").unwrap();
    let con = client.get_multiplexed_async_connection().await.unwrap();
    
    let state = Arc::new(AppState {
        redis: Mutex::new(con),
    });
    
    let app = Router::new()
        .route("/value", get(get_value))
        .with_state(state);
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

## 💡 最佳实践

| 场景 | 建议 |
|------|------|
| 连接管理 | 使用 `MultiplexedConnection` 或连接池 |
| Key 命名 | 用冒号分隔：`user:1001:profile` |
| 过期时间 | 缓存必须设过期，防止内存爆炸 |
| 序列化 | 复杂对象用 serde_json 序列化 |
| 错误处理 | 缓存失效不应导致请求失败 |

---

## 🎯 作业

1. 实现一个简单的限流器（每分钟最多 100 次请求）
2. 用 Redis Hash 存储用户 Session
3. 实现一个延迟队列（ZSET + 时间戳）

---

## 📖 参考资料

- [redis crate 文档](https://docs.rs/redis)
- [Redis 官方文档](https://redis.io/docs/)
- [Redis 命令参考](https://redis.io/commands/)

---

*下节课：连接池管理 (bb8/deadpool)*
