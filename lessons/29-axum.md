# 第29课：Axum Web 框架入门

> 🕐 授课时间：2026-02-16 09:00  
> 📍 地点：Telegram 群 Rust学习小组

---

## 什么是 Axum？

Axum 是 Tokio 团队出品的 Web 框架，特点：

- **零 Macro 魔法**：不像 Actix 那样大量使用宏
- **类型安全**：编译期检查路由参数
- **Tokio 原生**：和 Tokio 生态完美集成
- **Tower 兼容**：可以用 Tower 中间件生态

类比 PHP 世界：Axum ≈ 现代的 Laravel，但性能是它的百倍级别。

---

## 第一个 Axum 程序

```toml
# Cargo.toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
```

```rust
use axum::{Router, routing::get};

async fn hello() -> &'static str {
    "Hello, Axum!"
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(hello));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    
    println!("🚀 Server running at http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

运行：`cargo run`，访问 `http://localhost:3000`

---

## 路由与 HTTP 方法

```rust
use axum::{
    Router,
    routing::{get, post, put, delete},
};

async fn list_users() -> &'static str { "用户列表" }
async fn create_user() -> &'static str { "创建用户" }
async fn update_user() -> &'static str { "更新用户" }
async fn delete_user() -> &'static str { "删除用户" }

let app = Router::new()
    .route("/users", get(list_users).post(create_user))
    .route("/users/:id", put(update_user).delete(delete_user));
```

对比 Laravel：
```php
Route::get('/users', [UserController::class, 'index']);
Route::post('/users', [UserController::class, 'store']);
Route::put('/users/{id}', [UserController::class, 'update']);
Route::delete('/users/{id}', [UserController::class, 'destroy']);
```

---

## 路径参数（Path）

```rust
use axum::extract::Path;

// 单个参数
async fn get_user(Path(id): Path<u64>) -> String {
    format!("用户 ID: {}", id)
}

// 多个参数
async fn get_post(
    Path((user_id, post_id)): Path<(u64, u64)>
) -> String {
    format!("用户 {} 的文章 {}", user_id, post_id)
}

let app = Router::new()
    .route("/users/:id", get(get_user))
    .route("/users/:user_id/posts/:post_id", get(get_post));
```

对比 Laravel：
```php
Route::get('/users/{id}', function ($id) {
    return "用户 ID: $id";
});
```

---

## 查询参数（Query）

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct Pagination {
    page: Option<u32>,
    per_page: Option<u32>,
}

async fn list_users(Query(params): Query<Pagination>) -> String {
    let page = params.page.unwrap_or(1);
    let per_page = params.per_page.unwrap_or(20);
    format!("第 {} 页，每页 {} 条", page, per_page)
}

// GET /users?page=2&per_page=50
```

注意要添加 serde 依赖：
```toml
serde = { version = "1", features = ["derive"] }
```

---

## 请求体 JSON（Body）

```rust
use axum::Json;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Serialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

async fn create_user(Json(payload): Json<CreateUser>) -> Json<User> {
    let user = User {
        id: 1,
        name: payload.name,
        email: payload.email,
    };
    Json(user)
}
```

请求：
```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "张三", "email": "zhang@test.com"}'
```

响应：
```json
{"id":1,"name":"张三","email":"zhang@test.com"}
```

---

## 响应类型

Axum 的 handler 可以返回很多类型：

```rust
use axum::{
    response::{Html, IntoResponse, Response},
    http::StatusCode,
    Json,
};

// 纯文本
async fn text() -> &'static str {
    "Hello"
}

// HTML
async fn html() -> Html<&'static str> {
    Html("<h1>Hello</h1>")
}

// JSON
async fn json() -> Json<serde_json::Value> {
    Json(serde_json::json!({ "status": "ok" }))
}

// 状态码 + 响应体
async fn created() -> (StatusCode, Json<serde_json::Value>) {
    (
        StatusCode::CREATED,
        Json(serde_json::json!({ "id": 1 }))
    )
}

// 自定义错误
async fn not_found() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "资源不存在")
}
```

---

## 路由分组与嵌套

```rust
use axum::Router;

fn user_routes() -> Router {
    Router::new()
        .route("/", get(list_users).post(create_user))
        .route("/:id", get(get_user).put(update_user).delete(delete_user))
}

fn post_routes() -> Router {
    Router::new()
        .route("/", get(list_posts).post(create_post))
        .route("/:id", get(get_post))
}

let app = Router::new()
    .nest("/users", user_routes())
    .nest("/posts", post_routes());

// 生成的路由：
// GET  /users
// POST /users
// GET  /users/:id
// PUT  /users/:id
// DELETE /users/:id
// GET  /posts
// POST /posts
// GET  /posts/:id
```

对比 Laravel：
```php
Route::prefix('users')->group(function () {
    Route::get('/', [UserController::class, 'index']);
    Route::post('/', [UserController::class, 'store']);
    // ...
});
```

---

## 共享状态（State）

类似 Laravel 的依赖注入，但是编译期确定：

```rust
use axum::extract::State;
use std::sync::Arc;

// 应用状态
struct AppState {
    db_pool: String,  // 实际项目用 sqlx::PgPool
}

async fn get_users(State(state): State<Arc<AppState>>) -> String {
    format!("使用连接池: {}", state.db_pool)
}

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState {
        db_pool: "postgres://localhost/myapp".to_string(),
    });

    let app = Router::new()
        .route("/users", get(get_users))
        .with_state(state);  // 注入状态

    // ...
}
```

为什么用 `Arc`？因为状态要在多个请求间共享，`Arc` 是线程安全的引用计数。

---

## 中间件

Axum 兼容 Tower 中间件生态：

```rust
use axum::{
    Router,
    middleware::{self, Next},
    extract::Request,
    response::Response,
};
use std::time::Instant;

// 自定义中间件：请求计时
async fn timing_middleware(
    request: Request,
    next: Next,
) -> Response {
    let start = Instant::now();
    
    let response = next.run(request).await;
    
    let duration = start.elapsed();
    println!("请求耗时: {:?}", duration);
    
    response
}

let app = Router::new()
    .route("/", get(handler))
    .layer(middleware::from_fn(timing_middleware));
```

使用 tower-http 内置中间件：
```toml
tower-http = { version = "0.5", features = ["cors", "trace"] }
```

```rust
use tower_http::cors::{CorsLayer, Any};
use tower_http::trace::TraceLayer;

let app = Router::new()
    .route("/", get(handler))
    .layer(CorsLayer::new().allow_origin(Any))
    .layer(TraceLayer::new_for_http());
```

---

## 完整示例：简单的 Todo API

```rust
use axum::{
    Router,
    routing::{get, post},
    extract::{Path, State, Json},
    http::StatusCode,
};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};

#[derive(Clone, Serialize)]
struct Todo {
    id: u64,
    title: String,
    completed: bool,
}

#[derive(Deserialize)]
struct CreateTodo {
    title: String,
}

struct AppState {
    todos: Mutex<Vec<Todo>>,
    next_id: Mutex<u64>,
}

async fn list_todos(
    State(state): State<Arc<AppState>>
) -> Json<Vec<Todo>> {
    let todos = state.todos.lock().unwrap();
    Json(todos.clone())
}

async fn create_todo(
    State(state): State<Arc<AppState>>,
    Json(payload): Json<CreateTodo>,
) -> (StatusCode, Json<Todo>) {
    let mut next_id = state.next_id.lock().unwrap();
    let todo = Todo {
        id: *next_id,
        title: payload.title,
        completed: false,
    };
    *next_id += 1;
    
    state.todos.lock().unwrap().push(todo.clone());
    
    (StatusCode::CREATED, Json(todo))
}

async fn toggle_todo(
    State(state): State<Arc<AppState>>,
    Path(id): Path<u64>,
) -> Result<Json<Todo>, StatusCode> {
    let mut todos = state.todos.lock().unwrap();
    
    if let Some(todo) = todos.iter_mut().find(|t| t.id == id) {
        todo.completed = !todo.completed;
        Ok(Json(todo.clone()))
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState {
        todos: Mutex::new(vec![]),
        next_id: Mutex::new(1),
    });

    let app = Router::new()
        .route("/todos", get(list_todos).post(create_todo))
        .route("/todos/:id/toggle", post(toggle_todo))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    
    println!("🚀 Todo API at http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

## 💡 对比 Laravel

| 概念 | Laravel | Axum |
|------|---------|------|
| 路由定义 | `Route::get()` | `Router::new().route()` |
| 路径参数 | `{id}` | `:id` + `Path<T>` |
| 请求体 | `$request->input()` | `Json<T>` |
| 依赖注入 | 自动容器 | `State<T>` |
| 中间件 | `->middleware()` | `.layer()` |
| 路由分组 | `Route::prefix()` | `.nest()` |
| 响应 | `response()->json()` | `Json(value)` |

---

## 🧠 本课小结

1. **Axum** 是 Tokio 官方 Web 框架，类型安全、无宏魔法
2. **Handler** 是普通 async 函数，参数用 Extractor 提取
3. **Path/Query/Json** 分别提取路径、查询参数、请求体
4. **State** 共享应用状态（数据库连接池等）
5. **Router::nest** 实现路由分组
6. **layer** 添加中间件

---

## 📖 推荐资源

- [Axum 官方文档](https://docs.rs/axum)
- [Axum Examples](https://github.com/tokio-rs/axum/tree/main/examples)

---

*下节课：错误处理与数据库集成（sqlx）*
