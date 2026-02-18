# 第 52 课：数据验证 (validator)

> 日期：2026-02-19  
> 主题：使用 validator crate 进行声明式数据验证

---

## 📦 添加依赖

```toml
[dependencies]
validator = { version = "0.19", features = ["derive"] }
```

---

## 🎯 基础用法

```rust
use validator::{Validate, ValidationError};

#[derive(Debug, Validate)]
struct RegisterRequest {
    #[validate(length(min = 3, max = 20))]
    username: String,
    
    #[validate(email)]
    email: String,
    
    #[validate(length(min = 8), custom(function = "validate_password"))]
    password: String,
    
    #[validate(range(min = 18, max = 120))]
    age: u8,
}

// 自定义验证函数
fn validate_password(password: &str) -> Result<(), ValidationError> {
    // 必须包含数字
    if !password.chars().any(|c| c.is_numeric()) {
        return Err(ValidationError::new("password_no_number"));
    }
    // 必须包含大写字母
    if !password.chars().any(|c| c.is_uppercase()) {
        return Err(ValidationError::new("password_no_uppercase"));
    }
    Ok(())
}

fn main() {
    let req = RegisterRequest {
        username: "ab".to_string(),        // 太短！
        email: "not-an-email".to_string(), // 无效邮箱！
        password: "weakpass".to_string(),  // 没有数字和大写！
        age: 15,                           // 太小！
    };
    
    match req.validate() {
        Ok(()) => println!("验证通过"),
        Err(e) => println!("验证失败: {}", e),
    }
}
```

输出：
```
验证失败: age: Validation error: range [{}, ...]
email: Validation error: email [...]
password: Validation error: password_no_number [...]
username: Validation error: length [...]
```

---

## 🔧 常用验证器

| 验证器 | 用途 | 示例 |
|--------|------|------|
| `length` | 长度限制 | `#[validate(length(min = 1, max = 100))]` |
| `email` | 邮箱格式 | `#[validate(email)]` |
| `url` | URL 格式 | `#[validate(url)]` |
| `range` | 数值范围 | `#[validate(range(min = 0, max = 100))]` |
| `must_match` | 字段匹配 | `#[validate(must_match(other = "password"))]` |
| `contains` | 包含子串 | `#[validate(contains(pattern = "@"))]` |
| `regex` | 正则匹配 | `#[validate(regex(path = *RE_PHONE))]` |
| `custom` | 自定义函数 | `#[validate(custom(function = "my_fn"))]` |
| `nested` | 嵌套验证 | `#[validate(nested)]` |

---

## 📝 实战：注册表单验证

```rust
use once_cell::sync::Lazy;
use regex::Regex;
use validator::{Validate, ValidationError};

// 手机号正则（中国大陆）
static RE_PHONE: Lazy<Regex> = Lazy::new(|| {
    Regex::new(r"^1[3-9]\d{9}$").unwrap()
});

#[derive(Debug, Validate)]
struct SignupForm {
    #[validate(length(min = 2, max = 50, message = "用户名长度 2-50"))]
    username: String,
    
    #[validate(email(message = "邮箱格式不正确"))]
    email: String,
    
    #[validate(regex(path = *RE_PHONE, message = "手机号格式不正确"))]
    phone: String,
    
    #[validate(length(min = 8, message = "密码至少 8 位"))]
    password: String,
    
    #[validate(must_match(other = "password", message = "两次密码不一致"))]
    confirm_password: String,
    
    #[validate(nested)]
    address: Option<Address>,
}

#[derive(Debug, Validate)]
struct Address {
    #[validate(length(min = 1, message = "省份不能为空"))]
    province: String,
    
    #[validate(length(min = 1, message = "城市不能为空"))]
    city: String,
}
```

---

## 🌐 与 Axum 集成

```rust
use axum::{
    extract::Json,
    http::StatusCode,
    response::IntoResponse,
    routing::post,
    Router,
};
use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
struct CreateUserRequest {
    #[validate(length(min = 3, max = 20))]
    username: String,
    #[validate(email)]
    email: String,
}

#[derive(Serialize)]
struct ApiError {
    errors: Vec<String>,
}

async fn create_user(
    Json(payload): Json<CreateUserRequest>,
) -> impl IntoResponse {
    // 验证输入
    if let Err(validation_errors) = payload.validate() {
        let errors: Vec<String> = validation_errors
            .field_errors()
            .iter()
            .flat_map(|(field, errors)| {
                errors.iter().map(move |e| {
                    format!("{}: {}", field, 
                        e.message.clone().unwrap_or_default())
                })
            })
            .collect();
        
        return (
            StatusCode::BAD_REQUEST,
            Json(ApiError { errors }),
        ).into_response();
    }
    
    // 验证通过，处理业务逻辑
    (StatusCode::CREATED, Json(serde_json::json!({
        "message": "User created",
        "username": payload.username
    }))).into_response()
}
```

---

## 🎨 自定义错误消息

```rust
#[derive(Debug, Validate)]
struct Product {
    #[validate(length(
        min = 1, 
        max = 100, 
        message = "商品名称长度必须在 1-100 之间"
    ))]
    name: String,
    
    #[validate(range(
        min = 0.01, 
        max = 999999.99, 
        message = "价格必须在 0.01-999999.99 之间"
    ))]
    price: f64,
    
    #[validate(custom(
        function = "validate_sku",
        message = "SKU 格式不正确，应为 XXX-XXXX-XX"
    ))]
    sku: String,
}

fn validate_sku(sku: &str) -> Result<(), ValidationError> {
    let re = Regex::new(r"^[A-Z]{3}-\d{4}-[A-Z]{2}$").unwrap();
    if re.is_match(sku) {
        Ok(())
    } else {
        Err(ValidationError::new("invalid_sku"))
    }
}
```

---

## 💡 小技巧

### 1. Optional 字段只在有值时验证

```rust
#[derive(Validate)]
struct Profile {
    #[validate(url)]
    website: Option<String>,  // None 时跳过验证
}
```

### 2. 验证 Vec 中的每个元素

```rust
#[derive(Validate)]
struct Order {
    #[validate(length(min = 1))]
    #[validate]
    items: Vec<OrderItem>,
}

#[derive(Validate)]
struct OrderItem {
    #[validate(range(min = 1))]
    quantity: u32,
}
```

### 3. 条件验证（用 custom）

```rust
fn validate_premium_user(user: &User) -> Result<(), ValidationError> {
    if user.is_premium && user.subscription_id.is_none() {
        return Err(ValidationError::new("premium_needs_subscription"));
    }
    Ok(())
}
```

---

## 📋 课后练习

1. 为你的项目添加一个登录请求验证：邮箱格式 + 密码长度
2. 写一个自定义验证函数，检查用户名不能包含特殊字符
3. 实现 Axum 中间件，自动验证所有带 `#[derive(Validate)]` 的请求体

---

## 📚 参考资料

- [validator crate 文档](https://docs.rs/validator)
- [validator GitHub](https://github.com/Keats/validator)

---

*笔记整理：性奴001*
