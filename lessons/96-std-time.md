# 第 96 课：std::time — 时间与持续时间

> 日期：2026-02-24

时间处理是编程中的常见需求。Rust 标准库的 `std::time` 提供了简洁而安全的时间原语。

---

## 📦 模块概览

```rust
use std::time::{
    Duration,     // 时间段（多长）
    Instant,      // 单调时钟（计时用）
    SystemTime,   // 系统时钟（日期时间）
    UNIX_EPOCH,   // Unix 纪元（1970-01-01 00:00:00 UTC）
};
```

**两种时钟的区别：**
- `Instant`：只能往前走，用于测量**经过的时间**（性能计时）
- `SystemTime`：可能跳变（NTP 同步），用于表示**日期时间**

---

## ⏱️ Duration — 时间段

表示一段时间长度：

```rust
use std::time::Duration;

fn main() {
    // 创建 Duration
    let d1 = Duration::from_secs(60);        // 60 秒
    let d2 = Duration::from_millis(1500);    // 1500 毫秒
    let d3 = Duration::from_micros(1000);    // 1000 微秒
    let d4 = Duration::from_nanos(500);      // 500 纳秒
    
    // Rust 1.51+ 的便捷常量
    let five_sec = Duration::from_secs(5);
    let zero = Duration::ZERO;
    let max = Duration::MAX;
    
    // 读取
    println!("秒: {}", d1.as_secs());        // 60
    println!("毫秒: {}", d2.as_millis());    // 1500
    println!("小数秒: {}", d2.as_secs_f64()); // 1.5
    
    // 运算
    let sum = d1 + d2;
    let diff = d1 - Duration::from_secs(10);
    let doubled = d1 * 2;
    let halved = d1 / 2;
    
    // 比较
    if d1 > d2 {
        println!("d1 更长");
    }
}
```

类比 PHP：
```php
// PHP 用秒数或 DateInterval
$duration = 60; // 秒
$interval = new DateInterval('PT60S');
```

---

## ⚡ Instant — 单调时钟（性能计时）

`Instant` 是**单调递增**的时钟，专门用于测量时间间隔：

```rust
use std::time::Instant;

fn main() {
    // 记录开始时间
    let start = Instant::now();
    
    // 做一些耗时操作
    let sum: u64 = (1..=1_000_000).sum();
    
    // 计算耗时
    let elapsed = start.elapsed();
    println!("计算结果: {}", sum);
    println!("耗时: {:?}", elapsed);  // 例如: 2.3ms
    println!("耗时(毫秒): {}", elapsed.as_millis());
    
    // Instant 之间可以相减
    let later = Instant::now();
    let diff = later - start;
    println!("总耗时: {:?}", diff);
}
```

### 封装成计时器

```rust
use std::time::Instant;

struct Timer {
    label: String,
    start: Instant,
}

impl Timer {
    fn new(label: &str) -> Self {
        println!("[{}] 开始计时...", label);
        Timer {
            label: label.to_string(),
            start: Instant::now(),
        }
    }
}

impl Drop for Timer {
    fn drop(&mut self) {
        println!("[{}] 耗时: {:?}", self.label, self.start.elapsed());
    }
}

fn main() {
    {
        let _t = Timer::new("排序");
        let mut v: Vec<i32> = (0..100000).rev().collect();
        v.sort();
    } // 这里自动打印耗时
    
    {
        let _t = Timer::new("查找");
        let v: Vec<i32> = (0..100000).collect();
        let _ = v.binary_search(&50000);
    }
}
```

输出：
```
[排序] 开始计时...
[排序] 耗时: 8.123ms
[查找] 开始计时...
[查找] 耗时: 123µs
```

---

## 🕐 SystemTime — 系统时钟

`SystemTime` 表示系统时间（日期时间），可以转换为 Unix 时间戳：

```rust
use std::time::{SystemTime, UNIX_EPOCH, Duration};

fn main() {
    // 当前时间
    let now = SystemTime::now();
    
    // 转换为 Unix 时间戳
    let timestamp = now
        .duration_since(UNIX_EPOCH)
        .expect("时光倒流了？")
        .as_secs();
    println!("Unix 时间戳: {}", timestamp);
    
    // 从时间戳创建
    let time = UNIX_EPOCH + Duration::from_secs(1700000000);
    println!("时间: {:?}", time);
    
    // 时间比较
    let earlier = UNIX_EPOCH + Duration::from_secs(1000000000);
    if now > earlier {
        let diff = now.duration_since(earlier).unwrap();
        println!("距离 2001 年已过: {} 天", diff.as_secs() / 86400);
    }
}
```

### ⚠️ SystemTime 可能失败！

```rust
use std::time::{SystemTime, UNIX_EPOCH};

fn main() {
    let now = SystemTime::now();
    let future = now + std::time::Duration::from_secs(3600);
    
    // duration_since 返回 Result
    // 因为如果 self < other，就会失败
    match now.duration_since(future) {
        Ok(d) => println!("差: {:?}", d),
        Err(e) => {
            // SystemTimeError 包含反向的 duration
            println!("时间在未来！差: {:?}", e.duration());
        }
    }
    
    // 更安全的做法：用 checked 方法
    if let Some(earlier) = now.checked_sub(std::time::Duration::from_secs(60)) {
        println!("1 分钟前: {:?}", earlier);
    }
}
```

---

## 🔄 实战：限流器（Rate Limiter）

用 Instant 实现一个简单的令牌桶限流器：

```rust
use std::time::{Duration, Instant};

struct RateLimiter {
    capacity: u32,          // 桶容量
    tokens: u32,            // 当前令牌数
    refill_rate: u32,       // 每秒补充令牌数
    last_refill: Instant,   // 上次补充时间
}

impl RateLimiter {
    fn new(capacity: u32, refill_rate: u32) -> Self {
        RateLimiter {
            capacity,
            tokens: capacity,
            refill_rate,
            last_refill: Instant::now(),
        }
    }
    
    fn try_acquire(&mut self) -> bool {
        self.refill();
        
        if self.tokens > 0 {
            self.tokens -= 1;
            true
        } else {
            false
        }
    }
    
    fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill);
        
        // 计算应该补充的令牌数
        let new_tokens = (elapsed.as_secs_f64() * self.refill_rate as f64) as u32;
        
        if new_tokens > 0 {
            self.tokens = (self.tokens + new_tokens).min(self.capacity);
            self.last_refill = now;
        }
    }
}

fn main() {
    let mut limiter = RateLimiter::new(5, 2); // 容量5，每秒补2个
    
    for i in 1..=10 {
        if limiter.try_acquire() {
            println!("请求 {} ✅ 通过", i);
        } else {
            println!("请求 {} ❌ 被限流", i);
        }
        std::thread::sleep(Duration::from_millis(200));
    }
}
```

---

## ⏰ 实战：超时控制

```rust
use std::time::{Duration, Instant};

fn do_with_timeout<F, T>(timeout: Duration, mut f: F) -> Option<T>
where
    F: FnMut() -> Option<T>,
{
    let deadline = Instant::now() + timeout;
    
    loop {
        // 尝试执行
        if let Some(result) = f() {
            return Some(result);
        }
        
        // 检查超时
        if Instant::now() >= deadline {
            return None;
        }
        
        // 短暂休眠避免 CPU 空转
        std::thread::sleep(Duration::from_millis(10));
    }
}

fn main() {
    let mut counter = 0;
    
    // 模拟需要多次尝试才能成功的操作
    let result = do_with_timeout(Duration::from_secs(1), || {
        counter += 1;
        println!("尝试 #{}", counter);
        
        if counter >= 5 {
            Some("成功！")
        } else {
            None
        }
    });
    
    match result {
        Some(msg) => println!("结果: {}", msg),
        None => println!("超时！"),
    }
}
```

---

## 🧮 实战：简单基准测试

```rust
use std::time::{Duration, Instant};

fn benchmark<F>(name: &str, iterations: u32, mut f: F)
where
    F: FnMut(),
{
    // 预热
    for _ in 0..10 {
        f();
    }
    
    // 正式测量
    let start = Instant::now();
    for _ in 0..iterations {
        f();
    }
    let total = start.elapsed();
    
    let per_iter = total / iterations;
    println!(
        "{}: {} 次迭代, 总计 {:?}, 平均 {:?}/次",
        name, iterations, total, per_iter
    );
}

fn main() {
    benchmark("Vec push", 100_000, || {
        let mut v = Vec::new();
        for i in 0..1000 {
            v.push(i);
        }
    });
    
    benchmark("Vec with_capacity", 100_000, || {
        let mut v = Vec::with_capacity(1000);
        for i in 0..1000 {
            v.push(i);
        }
    });
}
```

---

## 💤 配合 thread::sleep

```rust
use std::time::{Duration, Instant};
use std::thread;

fn main() {
    println!("开始等待...");
    let start = Instant::now();
    
    // 休眠 500 毫秒
    thread::sleep(Duration::from_millis(500));
    
    println!("实际等待时间: {:?}", start.elapsed());
    // 通常会略多于 500ms（取决于系统调度）
}
```

---

## 📝 要点总结

| 类型 | 用途 | 特点 |
|------|------|------|
| `Duration` | 时间段 | 纯粹的时间长度 |
| `Instant` | 计时 | 单调递增，不会倒退 |
| `SystemTime` | 日期时间 | 可能跳变，可转时间戳 |

**什么时候用什么：**
- 测量代码性能 → `Instant`
- 文件修改时间 → `SystemTime`
- 设置超时时间 → `Duration`
- 日志打印时间 → `SystemTime`（或用 chrono 库格式化）

**注意事项：**
1. `Instant::elapsed()` 返回 `Duration`
2. `SystemTime::duration_since()` 返回 `Result`
3. 标准库不提供日期时间格式化（用 chrono 或 time crate）

---

*下节课：std::thread — 多线程编程*
