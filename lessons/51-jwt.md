# 第 51 课：JWT 认证 (jsonwebtoken)

上节课讲了密码哈希，这节课继续安全主题——JWT (JSON Web Token) 认证。

---

## 🎯 什么是 JWT？

JWT 是一种无状态的认证方案：
- **无状态**：服务器不存 session，token 自带用户信息
- **结构**：`header.payload.signature`（三段 base64）
- **用途**：API 认证、单点登录、微服务通信

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NSJ9.xxxxx
    header          .     payload      . signature
```

---

## 📦 添加依赖

```toml
[dependencies]
jsonwebtoken = "9"
serde = { version = "1", features = ["derive"] }
chrono = "0.4"
```

---

## 🔑 基本用法

### 定义 Claims 结构

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,        // subject (用户ID)
    exp: usize,         // 过期时间 (Unix timestamp)
    iat: usize,         // 签发时间
    role: String,       // 自定义字段
}
```

### 生成 Token

```rust
use jsonwebtoken::{encode, Header, EncodingKey};
use chrono::{Utc, Duration};

fn create_token(user_id: &str, role: &str, secret: &[u8]) -> String {
    let now = Utc::now();
    let claims = Claims {
        sub: user_id.to_string(),
        exp: (now + Duration::hours(24)).timestamp() as usize,
        iat: now.timestamp() as usize,
        role: role.to_string(),
    };
    
    encode(
        &Header::default(),  // 默认 HS256
        &claims,
        &EncodingKey::from_secret(secret),
    ).unwrap()
}
```

### 验证 Token

```rust
use jsonwebtoken::{decode, Validation, DecodingKey};

fn verify_token(token: &str, secret: &[u8]) -> Result<Claims, String> {
    decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret),
        &Validation::default(),
    )
    .map(|data| data.claims)
    .map_err(|e| e.to_string())
}
```

---

## ⚙️ 自定义验证规则

```rust
use jsonwebtoken::Validation;

let mut validation = Validation::default();

// 不验证过期时间（调试用）
validation.validate_exp = false;

// 必须包含某个 audience
validation.set_audience(&["my-app"]);

// 必须是某个 issuer 签发
validation.set_issuer(&["auth-server"]);

// 允许 5 分钟时钟偏差
validation.leeway = 300;
```

---

## 🔐 使用 RS256 (RSA 非对称加密)

生产环境推荐非对称加密——私钥签名，公钥验证：

```rust
use jsonwebtoken::{Algorithm, Header, EncodingKey, DecodingKey};

// 签名（用私钥）
let private_key = std::fs::read("private.pem")?;
let token = encode(
    &Header::new(Algorithm::RS256),
    &claims,
    &EncodingKey::from_rsa_pem(&private_key)?,
)?;

// 验证（用公钥）
let public_key = std::fs::read("public.pem")?;
let mut validation = Validation::new(Algorithm::RS256);
let token_data = decode::<Claims>(
    &token,
    &DecodingKey::from_rsa_pem(&public_key)?,
    &validation,
)?;
```

生成密钥对：
```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

---

## 🌐 Axum 集成示例

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode, header},
    response::IntoResponse,
    routing::{get, post},
    Json, Router,
};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use std::sync::LazyLock;

static SECRET: LazyLock<Vec<u8>> = LazyLock::new(|| {
    std::env::var("JWT_SECRET")
        .unwrap_or_else(|_| "dev-secret-change-me".into())
        .into_bytes()
});

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Claims {
    sub: String,
    exp: usize,
    role: String,
}

// 自定义 Extractor
struct AuthUser(Claims);

impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);
    
    async fn from_request_parts(parts: &mut Parts, _: &S) 
        -> Result<Self, Self::Rejection> 
    {
        let auth_header = parts.headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .ok_or((StatusCode::UNAUTHORIZED, "Missing token"))?;
        
        let token = auth_header
            .strip_prefix("Bearer ")
            .ok_or((StatusCode::UNAUTHORIZED, "Invalid format"))?;
        
        let claims = decode::<Claims>(
            token,
            &DecodingKey::from_secret(&SECRET),
            &Validation::default(),
        )
        .map_err(|_| (StatusCode::UNAUTHORIZED, "Invalid token"))?
        .claims;
        
        Ok(AuthUser(claims))
    }
}

// 受保护的路由
async fn protected(AuthUser(claims): AuthUser) -> impl IntoResponse {
    format!("Hello, {}! Role: {}", claims.sub, claims.role)
}
```

---

## 💡 最佳实践

| 实践 | 说明 |
|-----|-----|
| **短有效期** | Access token 15-60 分钟 |
| **Refresh Token** | 用于刷新 access token |
| **HTTPS** | Token 明文传输，必须加密 |
| **不存敏感数据** | Payload 是 base64，不是加密 |
| **RS256 生产** | 私钥签，公钥验，安全分离 |

---

## 🔄 Refresh Token 模式

```rust
#[derive(Serialize, Deserialize)]
struct TokenPair {
    access_token: String,   // 短期：15分钟
    refresh_token: String,  // 长期：7天，存数据库
}

// 刷新流程
async fn refresh(old_refresh: &str) -> Result<TokenPair, Error> {
    // 1. 验证 refresh token
    // 2. 检查是否在数据库白名单
    // 3. 生成新的 token pair
    // 4. 旧 refresh token 失效（rotate）
}
```

---

## ⚠️ 常见错误

```rust
// ❌ secret 太弱
let secret = b"123456";

// ✅ 至少 256 位随机
let secret = std::env::var("JWT_SECRET")?;

// ❌ 不验证算法（JWT alg 攻击）
// jsonwebtoken 默认安全，但要小心其他库

// ❌ Token 存 localStorage（XSS 风险）
// ✅ HttpOnly Cookie 更安全
```

---

## 📚 延伸阅读

- [jsonwebtoken crate](https://docs.rs/jsonwebtoken)
- [JWT.io](https://jwt.io/) - 在线调试工具
- [RFC 7519](https://tools.ietf.org/html/rfc7519) - JWT 规范

---

下节课预告：**Tower Middleware** —— 如何优雅地在 Axum 里做认证中间件！
