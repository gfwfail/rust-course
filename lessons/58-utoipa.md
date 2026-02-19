# 第 58 课：OpenAPI 文档生成 (utoipa)

> 授课时间：2026-02-20  
> 主题：用 utoipa 为 Axum API 自动生成 OpenAPI (Swagger) 文档

---

## 📚 今日主题

今天学习如何用 `utoipa` 为 Axum API 自动生成 OpenAPI (Swagger) 文档。

**为什么需要 API 文档？**
- 前后端协作必备
- 自动化测试工具可以直接读取
- 减少沟通成本，文档即代码

---

## 🛠️ 添加依赖

```toml
[dependencies]
utoipa = { version = "5", features = ["axum_extras"] }
utoipa-swagger-ui = { version = "8", features = ["axum"] }
utoipa-scalar = { version = "0.2", features = ["axum"] }
```

- `utoipa`: 核心库，提供宏来生成 OpenAPI spec
- `utoipa-swagger-ui`: Swagger UI 界面
- `utoipa-scalar`: 更现代的 Scalar UI（可选）

---

## 🎯 基础用法

### 1. 定义数据结构

```rust
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;

/// 用户信息
#[derive(Serialize, Deserialize, ToSchema)]
pub struct User {
    /// 用户 ID
    #[schema(example = 1)]
    pub id: i64,
    
    /// 用户名
    #[schema(example = "alice")]
    pub username: String,
    
    /// 邮箱地址
    #[schema(example = "alice@example.com")]
    pub email: String,
    
    /// 创建时间
    pub created_at: chrono::DateTime<chrono::Utc>,
}

/// 创建用户请求
#[derive(Deserialize, ToSchema)]
pub struct CreateUserRequest {
    #[schema(example = "bob", min_length = 3, max_length = 20)]
    pub username: String,
    
    #[schema(example = "bob@example.com")]
    pub email: String,
    
    #[schema(example = "password123", min_length = 8)]
    pub password: String,
}
```

**关键点：**
- `#[derive(ToSchema)]` 让结构体可被 OpenAPI 使用
- `#[schema(example = ...)]` 在文档中显示示例值
- 文档注释 `///` 会自动变成 description

---

### 2. 标注 API 路由

```rust
use axum::{extract::Path, Json};
use utoipa::OpenApi;

/// 获取用户列表
#[utoipa::path(
    get,
    path = "/api/users",
    tag = "users",
    responses(
        (status = 200, description = "成功获取用户列表", body = Vec<User>),
        (status = 500, description = "服务器错误")
    )
)]
pub async fn list_users() -> Json<Vec<User>> {
    // 实现...
    Json(vec![])
}

/// 根据 ID 获取用户
#[utoipa::path(
    get,
    path = "/api/users/{id}",
    tag = "users",
    params(
        ("id" = i64, Path, description = "用户 ID")
    ),
    responses(
        (status = 200, description = "成功", body = User),
        (status = 404, description = "用户不存在")
    )
)]
pub async fn get_user(Path(id): Path<i64>) -> Json<User> {
    // 实现...
    todo!()
}

/// 创建新用户
#[utoipa::path(
    post,
    path = "/api/users",
    tag = "users",
    request_body = CreateUserRequest,
    responses(
        (status = 201, description = "创建成功", body = User),
        (status = 400, description = "请求参数错误"),
        (status = 409, description = "用户名已存在")
    )
)]
pub async fn create_user(
    Json(req): Json<CreateUserRequest>
) -> Json<User> {
    // 实现...
    todo!()
}
```

---

### 3. 组装 OpenAPI 文档

```rust
use utoipa::OpenApi;

#[derive(OpenApi)]
#[openapi(
    info(
        title = "My API",
        version = "1.0.0",
        description = "这是一个示例 API",
        contact(
            name = "API Support",
            email = "support@example.com"
        )
    ),
    tags(
        (name = "users", description = "用户管理"),
        (name = "orders", description = "订单管理")
    ),
    paths(
        list_users,
        get_user,
        create_user,
    ),
    components(
        schemas(User, CreateUserRequest)
    )
)]
pub struct ApiDoc;
```

---

### 4. 挂载 Swagger UI

```rust
use axum::{Router, routing::get};
use utoipa_swagger_ui::SwaggerUi;

#[tokio::main]
async fn main() {
    let app = Router::new()
        // 业务路由
        .route("/api/users", get(list_users).post(create_user))
        .route("/api/users/{id}", get(get_user))
        // Swagger UI - 访问 /swagger-ui
        .merge(
            SwaggerUi::new("/swagger-ui")
                .url("/api-docs/openapi.json", ApiDoc::openapi())
        );
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
        
    println!("Swagger UI: http://localhost:3000/swagger-ui");
    axum::serve(listener, app).await.unwrap();
}
```

运行后访问 `http://localhost:3000/swagger-ui` 就能看到漂亮的 API 文档了！

---

## 🔐 添加认证支持

```rust
use utoipa::openapi::security::{HttpAuthScheme, HttpBuilder, SecurityScheme};

#[derive(OpenApi)]
#[openapi(
    // ... 其他配置
    components(
        schemas(User, CreateUserRequest),
        // 定义安全方案
        security_schemes(
            ("bearer_auth" = SecurityScheme::Http(
                HttpBuilder::new()
                    .scheme(HttpAuthScheme::Bearer)
                    .bearer_format("JWT")
                    .build()
            ))
        )
    ),
    // 全局应用安全方案
    security(
        ("bearer_auth" = [])
    )
)]
pub struct ApiDoc;
```

在路由上指定安全要求：

```rust
#[utoipa::path(
    get,
    path = "/api/users/me",
    tag = "users",
    security(
        ("bearer_auth" = [])
    ),
    responses(
        (status = 200, body = User),
        (status = 401, description = "未授权")
    )
)]
pub async fn get_current_user() -> Json<User> {
    todo!()
}
```

---

## 📊 更多 Schema 技巧

### 枚举

```rust
#[derive(Serialize, ToSchema)]
#[serde(rename_all = "snake_case")]
pub enum OrderStatus {
    Pending,
    Paid,
    Shipped,
    Completed,
    Cancelled,
}
```

### 嵌套对象

```rust
#[derive(Serialize, ToSchema)]
pub struct Order {
    pub id: i64,
    pub status: OrderStatus,
    pub items: Vec<OrderItem>,
    pub user: User,  // 嵌套引用
}
```

### 可选字段

```rust
#[derive(Deserialize, ToSchema)]
pub struct UpdateUserRequest {
    #[schema(nullable)]
    pub username: Option<String>,
    
    #[schema(nullable)]
    pub email: Option<String>,
}
```

---

## 🎨 使用 Scalar UI（更现代）

Scalar 是比 Swagger UI 更现代的替代品：

```rust
use utoipa_scalar::{Scalar, Servable};

let app = Router::new()
    .route("/api/users", get(list_users))
    // Scalar UI
    .merge(Scalar::with_url("/scalar", ApiDoc::openapi()));
```

---

## 💡 实战建议

1. **文档和代码同步**
   - Schema 从结构体生成，不会过时
   - 路由注解和实际代码在一起

2. **分模块组织**
   ```rust
   // users/mod.rs
   pub mod routes;
   pub mod schemas;
   
   // 在 main.rs 组合
   #[derive(OpenApi)]
   #[openapi(
       paths(users::routes::list, users::routes::create),
       components(schemas(users::schemas::User))
   )]
   ```

3. **生产环境可关闭**
   ```rust
   #[cfg(debug_assertions)]
   app.merge(SwaggerUi::new("/swagger-ui").url(...))
   ```

4. **导出 JSON/YAML**
   ```rust
   // 导出 OpenAPI spec
   let spec = ApiDoc::openapi().to_pretty_json()?;
   std::fs::write("openapi.json", spec)?;
   ```

---

## 📝 课后练习

1. 给你现有的 Axum 项目添加 utoipa
2. 为所有 API 路由添加文档注解
3. 尝试添加 JWT 认证的安全方案
4. 比较 Swagger UI 和 Scalar UI 的体验

---

## 🔗 相关资源

- [utoipa 官方文档](https://docs.rs/utoipa/latest/utoipa/)
- [utoipa 示例项目](https://github.com/juhaku/utoipa/tree/master/examples)
- [OpenAPI 规范](https://swagger.io/specification/)

---

*下节课预告：Session 会话管理 - 学习如何用 tower-sessions 实现服务端会话*
