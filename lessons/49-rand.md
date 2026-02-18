# 第 49 课：随机数生成 (rand)

> 日期：2026-02-18  
> 主题：rand crate 的使用

---

## 📚 为什么需要 rand？

在 Web 开发中，随机数无处不在：
- 生成验证码、Token
- 随机打乱数组顺序
- 抽奖、随机选择
- 测试数据生成
- 密码学安全随机数

Rust 标准库**不包含**随机数功能，需要用 `rand` crate。

---

## 🚀 基础用法

**添加依赖：**
```toml
[dependencies]
rand = "0.9"
```

**生成随机数：**
```rust
use rand::Rng;

fn main() {
    // 获取线程本地随机数生成器
    let mut rng = rand::rng();
    
    // 生成 i32 随机数
    let n: i32 = rng.random();
    println!("随机 i32: {}", n);
    
    // 生成范围内的随机数 [1, 100]
    let n: i32 = rng.random_range(1..=100);
    println!("1-100: {}", n);
    
    // 生成 bool
    let b: bool = rng.random();
    println!("随机 bool: {}", b);
    
    // 生成 f64 [0.0, 1.0)
    let f: f64 = rng.random();
    println!("随机 f64: {}", f);
}
```

---

## 🎯 常用场景

### 1️⃣ 生成随机字符串（验证码）

```rust
use rand::Rng;

fn gen_code(len: usize) -> String {
    const CHARSET: &[u8] = b"ABCDEFGHJKLMNPQRSTUVWXYZ23456789";
    let mut rng = rand::rng();
    
    (0..len)
        .map(|_| {
            let idx = rng.random_range(0..CHARSET.len());
            CHARSET[idx] as char
        })
        .collect()
}

fn main() {
    println!("验证码: {}", gen_code(6));
    // 输出: 验证码: X7KP3N
}
```

### 2️⃣ 随机选择元素

```rust
use rand::prelude::*;

fn main() {
    let items = vec!["苹果", "香蕉", "橙子", "葡萄"];
    let mut rng = rand::rng();
    
    // 随机选一个
    if let Some(item) = items.choose(&mut rng) {
        println!("抽中: {}", item);
    }
    
    // 随机选多个（不重复）
    let picked: Vec<_> = items.choose_multiple(&mut rng, 2).collect();
    println!("选中: {:?}", picked);
}
```

### 3️⃣ 打乱顺序 (Shuffle)

```rust
use rand::prelude::*;

fn main() {
    let mut cards = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    let mut rng = rand::rng();
    
    cards.shuffle(&mut rng);
    println!("洗牌后: {:?}", cards);
    // 输出: 洗牌后: [7, 3, 10, 1, 5, 9, 2, 8, 4, 6]
}
```

### 4️⃣ 按权重随机选择

```rust
use rand::prelude::*;

fn main() {
    let items = ["普通", "稀有", "史诗", "传说"];
    let weights = [70, 20, 8, 2]; // 概率权重
    
    let mut rng = rand::rng();
    
    // 创建带权重的分布
    let dist = rand::distr::weighted::WeightedIndex::new(&weights)
        .unwrap();
    
    // 抽取 10 次
    for _ in 0..10 {
        let idx = dist.sample(&mut rng);
        println!("抽到: {}", items[idx]);
    }
}
```

---

## 🔐 密码学安全随机数

普通随机数**不适合**用于：密码、Token、密钥

```rust
use rand::rngs::OsRng;
use rand::Rng;

fn gen_secure_token() -> String {
    let mut rng = OsRng;  // 使用操作系统的安全随机源
    
    let bytes: [u8; 32] = rng.random();
    
    // 转为 hex 字符串
    bytes.iter()
        .map(|b| format!("{:02x}", b))
        .collect()
}

fn main() {
    let token = gen_secure_token();
    println!("安全 Token: {}", token);
    // 输出: 安全 Token: a1b2c3d4e5f6...（64 位 hex）
}
```

---

## 📊 分布类型

```rust
use rand::prelude::*;
use rand::distr::{Uniform, Standard};

fn main() {
    let mut rng = rand::rng();
    
    // 均匀分布
    let uniform = Uniform::new(1, 10).unwrap();
    let n: i32 = uniform.sample(&mut rng);
    
    // 标准分布 (0.0 - 1.0)
    let f: f64 = Standard.sample(&mut rng);
    
    println!("均匀分布: {}, 标准分布: {}", n, f);
}
```

---

## 🆚 与其他语言对比

| 场景 | PHP | Rust |
|------|-----|------|
| 随机整数 | `rand(1, 100)` | `rng.random_range(1..=100)` |
| 随机浮点 | `mt_rand() / mt_getrandmax()` | `rng.random::<f64>()` |
| 打乱数组 | `shuffle($arr)` | `arr.shuffle(&mut rng)` |
| 随机选择 | `$arr[array_rand($arr)]` | `arr.choose(&mut rng)` |

---

## 💡 最佳实践

1. **复用 rng** - 不要每次都创建新的随机数生成器
2. **敏感场景用 OsRng** - 密码、Token 必须用密码学安全的随机源
3. **可重现测试** - 需要固定种子时用 `StdRng::seed_from_u64(12345)`

```rust
use rand::{SeedableRng, rngs::StdRng, Rng};

fn main() {
    // 固定种子，结果可重现（用于测试）
    let mut rng = StdRng::seed_from_u64(42);
    
    println!("{}", rng.random_range(1..=100)); // 总是相同结果
}
```

---

## 🏠 课后练习

1. 写一个函数生成 6 位数字验证码
2. 实现一个简单的抽奖系统（带权重）
3. 用固定种子生成可重现的测试数据

---

## 📖 参考资源

- 官方文档: https://docs.rs/rand
- Crates.io: https://crates.io/crates/rand
- The Rand Book: https://rust-random.github.io/book/
