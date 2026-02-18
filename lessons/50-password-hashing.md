# 第 50 课：密码哈希 (argon2 / bcrypt)

> 日期：2026-02-19  
> 主题：安全存储用户密码

---

## 为什么需要密码哈希？

存储用户密码时，**绝对不能**存明文！正确做法是存储密码的**单向哈希值**。

```
用户密码: "123456"
    ↓ 哈希函数（不可逆）
存储值: "$argon2id$v=19$m=19456,t=2,p=1$..."
```

### Laravel 对比

```php
// Laravel 用 Hash facade
Hash::make('password');         // 生成哈希
Hash::check('password', $hash); // 验证
```

Rust 中我们用 `argon2` 或 `bcrypt` crate。

---

## 算法选择

| 算法 | 推荐度 | 说明 |
|------|--------|------|
| **Argon2** | ⭐⭐⭐ | 2015年密码哈希竞赛冠军，最安全 |
| **bcrypt** | ⭐⭐ | 成熟稳定，广泛使用 |
| scrypt | ⭐⭐ | 内存密集型，也不错 |
| MD5/SHA | ❌ | **禁止用于密码！** 太快易被暴力破解 |

**结论：新项目优先用 Argon2，老项目兼容用 bcrypt**

---

## 安装依赖

```toml
# Cargo.toml
[dependencies]
argon2 = "0.5"        # Argon2 哈希
password-hash = "0.5" # 通用密码哈希接口
rand_core = { version = "0.6", features = ["std"] }
```

---

## Argon2 基础用法

```rust
use argon2::{
    password_hash::{
        rand_core::OsRng,
        PasswordHash, PasswordHasher, PasswordVerifier, SaltString
    },
    Argon2
};

fn main() {
    let password = b"super_secret_password";
    
    // 1. 生成随机盐
    let salt = SaltString::generate(&mut OsRng);
    
    // 2. 哈希密码
    let argon2 = Argon2::default();
    let password_hash = argon2
        .hash_password(password, &salt)
        .expect("哈希失败")
        .to_string();
    
    println!("哈希值: {}", password_hash);
    // 输出类似: $argon2id$v=19$m=19456,t=2,p=1$base64salt$base64hash
    
    // 3. 验证密码
    let parsed_hash = PasswordHash::new(&password_hash)
        .expect("解析哈希失败");
    
    let is_valid = argon2
        .verify_password(b"super_secret_password", &parsed_hash)
        .is_ok();
    
    println!("密码正确: {}", is_valid); // true
    
    // 错误密码
    let is_wrong = argon2
        .verify_password(b"wrong_password", &parsed_hash)
        .is_ok();
    
    println!("错误密码: {}", is_wrong); // false
}
```

---

## 封装成实用函数

```rust
use argon2::{
    password_hash::{
        rand_core::OsRng,
        PasswordHash, PasswordHasher, PasswordVerifier, SaltString
    },
    Argon2
};

/// 哈希密码（注册时用）
pub fn hash_password(password: &str) -> Result<String, argon2::password_hash::Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    
    Ok(argon2
        .hash_password(password.as_bytes(), &salt)?
        .to_string())
}

/// 验证密码（登录时用）
pub fn verify_password(password: &str, hash: &str) -> bool {
    let Ok(parsed_hash) = PasswordHash::new(hash) else {
        return false;
    };
    
    Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok()
}

// 使用示例
fn main() {
    // 用户注册
    let hash = hash_password("my_password").unwrap();
    println!("存入数据库: {}", hash);
    
    // 用户登录
    if verify_password("my_password", &hash) {
        println!("✅ 登录成功");
    } else {
        println!("❌ 密码错误");
    }
}
```

---

## 自定义 Argon2 参数

```rust
use argon2::{Algorithm, Argon2, Params, Version};

// 生产环境推荐配置
let params = Params::new(
    19456,  // m_cost: 内存大小 (KB)，约 19MB
    2,      // t_cost: 迭代次数
    1,      // p_cost: 并行度
    None,   // 输出长度，None 使用默认 32 字节
).expect("参数无效");

let argon2 = Argon2::new(
    Algorithm::Argon2id, // 推荐使用 Argon2id
    Version::V0x13,      // 版本 19
    params,
);

// 然后用这个 argon2 实例去哈希
```

### 参数调优建议

- **m_cost**：越大越安全，但消耗更多内存
- **t_cost**：越大越安全，但耗时更长
- **目标**：单次哈希耗时 0.3-1 秒左右

---

## bcrypt 用法（备选）

```toml
[dependencies]
bcrypt = "0.15"
```

```rust
use bcrypt::{hash, verify, DEFAULT_COST};

fn main() {
    let password = "my_password";
    
    // 哈希（DEFAULT_COST = 12）
    let hashed = hash(password, DEFAULT_COST).unwrap();
    println!("哈希: {}", hashed);
    // 输出: $2b$12$...
    
    // 验证
    let valid = verify(password, &hashed).unwrap();
    println!("验证: {}", valid);
}
```

bcrypt 的 API 更简单，但 Argon2 安全性更高。

---

## 在 Axum 中使用

```rust
use axum::{Json, Router, routing::post};
use serde::{Deserialize, Serialize};

mod password {
    // 上面封装的 hash_password 和 verify_password
}

#[derive(Deserialize)]
struct RegisterRequest {
    email: String,
    password: String,
}

#[derive(Deserialize)]
struct LoginRequest {
    email: String,
    password: String,
}

#[derive(Serialize)]
struct AuthResponse {
    success: bool,
    message: String,
}

async fn register(Json(req): Json<RegisterRequest>) -> Json<AuthResponse> {
    // 1. 哈希密码
    let password_hash = match password::hash_password(&req.password) {
        Ok(h) => h,
        Err(_) => return Json(AuthResponse {
            success: false,
            message: "密码处理失败".into(),
        }),
    };
    
    // 2. 存入数据库（password_hash 字段）
    // db.insert_user(&req.email, &password_hash).await?;
    
    Json(AuthResponse {
        success: true,
        message: "注册成功".into(),
    })
}

async fn login(Json(req): Json<LoginRequest>) -> Json<AuthResponse> {
    // 1. 从数据库查用户
    // let user = db.find_user(&req.email).await?;
    let stored_hash = "$argon2id$v=19$..."; // 假设从 DB 读取
    
    // 2. 验证密码
    if password::verify_password(&req.password, stored_hash) {
        Json(AuthResponse {
            success: true,
            message: "登录成功".into(),
        })
    } else {
        Json(AuthResponse {
            success: false,
            message: "密码错误".into(),
        })
    }
}
```

---

## 💡 要点总结

1. **永远不存明文密码**
2. **优先选择 Argon2id**，bcrypt 作为备选
3. **不要用 MD5/SHA** 做密码哈希
4. 哈希值自带盐，不需要单独存盐
5. 验证时用恒定时间比较（库已处理）
6. 调整参数使哈希耗时 0.3-1 秒

---

## 下一课预告

JWT 认证 (jsonwebtoken)，配合今天的密码哈希，完整实现用户认证系统！
