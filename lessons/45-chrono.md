# 第45课：日期时间处理 (chrono)

> 日期：2026-02-18

---

## 📚 概述

在 Web 开发中，处理日期时间是家常便饭：
- 用户注册时间、订单创建时间
- 定时任务、过期检查
- 时区转换、格式化输出

Rust 标准库有 `std::time`，但功能有限。**chrono** 是 Rust 最流行的日期时间库。

## 🔧 添加依赖

```toml
[dependencies]
chrono = { version = "0.4", features = ["serde"] }
```

---

## 📖 核心类型

chrono 有几个核心类型，理解它们是关键：

```rust
use chrono::{DateTime, Utc, Local, NaiveDate, NaiveDateTime, NaiveTime};

// 1. Naive 类型 - 不带时区信息
let naive_date = NaiveDate::from_ymd_opt(2026, 2, 18).unwrap();
let naive_time = NaiveTime::from_hms_opt(9, 0, 0).unwrap();
let naive_dt = NaiveDateTime::new(naive_date, naive_time);
println!("Naive: {}", naive_dt); // 2026-02-18 09:00:00

// 2. DateTime<Utc> - UTC 时间
let utc_now: DateTime<Utc> = Utc::now();
println!("UTC: {}", utc_now); // 2026-02-17T22:00:00Z

// 3. DateTime<Local> - 本地时间
let local_now: DateTime<Local> = Local::now();
println!("Local: {}", local_now); // 2026-02-18T09:00:00+11:00
```

### Naive vs DateTime 的区别

| 类型 | 带时区 | 使用场景 |
|------|--------|----------|
| `NaiveDateTime` | ❌ | 生日、纪念日（不关心时区） |
| `DateTime<Utc>` | ✅ | 数据库存储、API 传输 |
| `DateTime<Local>` | ✅ | 用户界面显示 |

---

## 🛠️ 常用操作

### 1. 创建时间

```rust
use chrono::{Utc, Local, TimeZone, NaiveDate};

// 当前时间
let now = Utc::now();
let local_now = Local::now();

// 指定时间
let dt = Utc.with_ymd_and_hms(2026, 2, 18, 9, 0, 0).unwrap();

// 从 NaiveDateTime 转 DateTime
let naive = NaiveDate::from_ymd_opt(2026, 2, 18)
    .unwrap()
    .and_hms_opt(9, 0, 0)
    .unwrap();
let utc_dt = Utc.from_utc_datetime(&naive);
```

### 2. 格式化输出

```rust
use chrono::{Utc, Local};

let now = Local::now();

// 内置格式
println!("{}", now.to_rfc2822());  // Wed, 18 Feb 2026 09:00:00 +1100
println!("{}", now.to_rfc3339());  // 2026-02-18T09:00:00+11:00

// 自定义格式
println!("{}", now.format("%Y-%m-%d"));           // 2026-02-18
println!("{}", now.format("%Y年%m月%d日"));        // 2026年02月18日
println!("{}", now.format("%H:%M:%S"));           // 09:00:00
println!("{}", now.format("%Y-%m-%d %H:%M:%S"));  // 2026-02-18 09:00:00
```

### 常用格式符

| 符号 | 含义 | 示例 |
|------|------|------|
| `%Y` | 4位年 | 2026 |
| `%m` | 月 (01-12) | 02 |
| `%d` | 日 (01-31) | 18 |
| `%H` | 时 (00-23) | 09 |
| `%M` | 分 (00-59) | 30 |
| `%S` | 秒 (00-59) | 45 |
| `%a` | 星期缩写 | Wed |
| `%A` | 星期全称 | Wednesday |
| `%Z` | 时区 | +11:00 |

### 3. 解析字符串

```rust
use chrono::{DateTime, Utc, NaiveDateTime};

// 解析 RFC 3339 格式
let dt = DateTime::parse_from_rfc3339("2026-02-18T09:00:00+11:00").unwrap();
println!("Parsed: {}", dt);

// 解析自定义格式
let naive = NaiveDateTime::parse_from_str(
    "2026-02-18 09:00:00",
    "%Y-%m-%d %H:%M:%S"
).unwrap();
println!("Naive: {}", naive);

// NaiveDateTime 转 DateTime<Utc>
let utc_dt: DateTime<Utc> = naive.and_utc();
```

---

## ⏰ 时间运算

```rust
use chrono::{Utc, Duration};

let now = Utc::now();

// 加减时间
let tomorrow = now + Duration::days(1);
let yesterday = now - Duration::days(1);
let later = now + Duration::hours(3) + Duration::minutes(30);

// 时间差
let diff = tomorrow - now;
println!("相差 {} 秒", diff.num_seconds());  // 86400
println!("相差 {} 小时", diff.num_hours());  // 24

// 比较
if tomorrow > now {
    println!("tomorrow is in the future");
}
```

---

## 🌍 时区转换

```rust
use chrono::{Utc, Local, FixedOffset, TimeZone, DateTime};

let utc_now = Utc::now();
println!("UTC: {}", utc_now);

// UTC 转本地时间
let local: DateTime<Local> = utc_now.with_timezone(&Local);
println!("Local: {}", local);

// 指定时区 (UTC+8 北京时间)
let beijing_tz = FixedOffset::east_opt(8 * 3600).unwrap();
let beijing_time = utc_now.with_timezone(&beijing_tz);
println!("Beijing: {}", beijing_time);

// 指定时区 (UTC+11 悉尼时间)
let sydney_tz = FixedOffset::east_opt(11 * 3600).unwrap();
let sydney_time = utc_now.with_timezone(&sydney_tz);
println!("Sydney: {}", sydney_time);
```

---

## 📦 与 Serde 集成

这是 Web 开发中最常用的场景：

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    name: String,
    // 默认序列化为 RFC 3339 格式
    created_at: DateTime<Utc>,
}

fn main() {
    let user = User {
        name: "Alice".to_string(),
        created_at: Utc::now(),
    };
    
    let json = serde_json::to_string(&user).unwrap();
    println!("{}", json);
    // {"name":"Alice","created_at":"2026-02-18T09:00:00Z"}
}
```

### 自定义序列化格式

```rust
use chrono::{DateTime, Utc, NaiveDateTime};
use serde::{self, Deserialize, Deserializer, Serializer};

mod date_format {
    use super::*;

    const FORMAT: &str = "%Y-%m-%d %H:%M:%S";

    pub fn serialize<S>(date: &DateTime<Utc>, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let s = date.format(FORMAT).to_string();
        serializer.serialize_str(&s)
    }

    pub fn deserialize<'de, D>(deserializer: D) -> Result<DateTime<Utc>, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        NaiveDateTime::parse_from_str(&s, FORMAT)
            .map(|dt| dt.and_utc())
            .map_err(serde::de::Error::custom)
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct Order {
    id: u64,
    #[serde(with = "date_format")]
    created_at: DateTime<Utc>,
}
```

---

## 🎯 实战：判断订单是否过期

```rust
use chrono::{DateTime, Utc, Duration};

struct Order {
    id: u64,
    created_at: DateTime<Utc>,
    expires_in_hours: i64,
}

impl Order {
    fn is_expired(&self) -> bool {
        let expiry = self.created_at + Duration::hours(self.expires_in_hours);
        Utc::now() > expiry
    }
    
    fn time_remaining(&self) -> Option<Duration> {
        let expiry = self.created_at + Duration::hours(self.expires_in_hours);
        let remaining = expiry - Utc::now();
        if remaining > Duration::zero() {
            Some(remaining)
        } else {
            None
        }
    }
}

fn main() {
    let order = Order {
        id: 1,
        created_at: Utc::now() - Duration::hours(23),
        expires_in_hours: 24,
    };
    
    if order.is_expired() {
        println!("订单已过期");
    } else if let Some(remaining) = order.time_remaining() {
        println!("剩余 {} 分钟", remaining.num_minutes());
    }
}
```

---

## 📝 chrono vs time

Rust 还有另一个日期库叫 `time`，两者对比：

| 特性 | chrono | time |
|------|--------|------|
| 流行度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| API 风格 | 更丰富 | 更简洁 |
| 时区支持 | FixedOffset | 更完整的 IANA 时区 |
| Serde 集成 | ✅ | ✅ |
| SQLx 支持 | ✅ | ✅ |

**建议**：大多数项目用 chrono 就够了，生态更成熟。

---

## 💡 最佳实践

1. **数据库存 UTC**：始终用 `DateTime<Utc>` 存储，显示时再转本地时区
2. **API 用 RFC 3339**：`2026-02-18T09:00:00Z` 是标准格式
3. **比较用 UTC**：避免时区问题导致的 bug
4. **Duration 别用 months**：月份长度不固定，用 days 更安全

---

## 🏠 课后作业

1. 写一个函数，计算两个日期之间相差多少天
2. 实现一个"倒计时"功能：输入未来时间，显示还有多久
3. 解析 `"18/02/2026"` 格式的日期字符串

---

*下节课：正则表达式 (regex)*
