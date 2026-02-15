# 第8课：Enum 实战场景

日常工作中什么情况会用到 Enum？这里用 Web 开发的实际场景讲解。

## 1. API 响应状态

```rust
enum ApiResponse<T> {
    Success(T),
    Error { code: u32, message: String },
    Loading,
}

// 处理响应
match fetch_user(id) {
    ApiResponse::Success(user) => json!(user),
    ApiResponse::Error { code, message } => {
        json!({ "error": message, "code": code })
    }
    ApiResponse::Loading => json!({ "status": "loading" }),
}
```

**对比 Laravel**：你可能用数组 + status 字段，但容易漏处理某种情况。Enum 编译器帮你查。

## 2. 订单状态机

```rust
enum OrderStatus {
    Pending,
    Paid { paid_at: DateTime, amount: f64 },
    Shipped { tracking_number: String },
    Delivered { delivered_at: DateTime },
    Cancelled { reason: String },
    Refunded { refund_amount: f64 },
}

fn can_cancel(status: &OrderStatus) -> bool {
    match status {
        OrderStatus::Pending => true,
        OrderStatus::Paid { .. } => true,  // 付款了还能取消
        _ => false,  // 已发货就不能了
    }
}
```

**Laravel 里**：你可能用字符串 `"pending"`, `"paid"`，但没人阻止你写成 `"penidng"`（typo）。

## 3. 用户权限/角色

```rust
enum Role {
    Guest,
    User { user_id: u64 },
    Admin { user_id: u64, permissions: Vec<String> },
    SuperAdmin,
}

fn can_delete_post(role: &Role, post_owner_id: u64) -> bool {
    match role {
        Role::SuperAdmin => true,
        Role::Admin { permissions, .. } => {
            permissions.contains(&"delete_post".to_string())
        }
        Role::User { user_id } => *user_id == post_owner_id,
        Role::Guest => false,
    }
}
```

## 4. 支付方式

```rust
enum PaymentMethod {
    CreditCard { last_four: String, brand: String },
    PayPal { email: String },
    Crypto { wallet_address: String, chain: String },
    BankTransfer { account: String },
}

fn process_payment(method: PaymentMethod, amount: f64) -> Result<(), PaymentError> {
    match method {
        PaymentMethod::CreditCard { last_four, .. } => {
            // 调用 Stripe
        }
        PaymentMethod::PayPal { email } => {
            // 调用 PayPal API
        }
        PaymentMethod::Crypto { wallet_address, chain } => {
            // 调用链上合约
        }
        PaymentMethod::BankTransfer { account } => {
            // 生成转账单
        }
    }
}
```

## 5. 消息/事件类型

```rust
enum WebSocketMessage {
    Chat { room_id: u64, content: String },
    Join { room_id: u64 },
    Leave { room_id: u64 },
    Typing { room_id: u64 },
    Ping,
    Pong,
}

// 处理 WebSocket 消息
fn handle_message(msg: WebSocketMessage) {
    match msg {
        WebSocketMessage::Chat { room_id, content } => {
            broadcast_to_room(room_id, content);
        }
        WebSocketMessage::Join { room_id } => {
            add_user_to_room(room_id);
        }
        // ...
    }
}
```

## 6. 表单验证错误

```rust
enum ValidationError {
    Required { field: String },
    TooShort { field: String, min: usize },
    TooLong { field: String, max: usize },
    InvalidEmail { field: String },
    InvalidFormat { field: String, expected: String },
}

fn validate_user(input: &UserInput) -> Result<(), Vec<ValidationError>> {
    let mut errors = vec![];
    
    if input.name.is_empty() {
        errors.push(ValidationError::Required { 
            field: "name".into() 
        });
    }
    
    if input.name.len() < 2 {
        errors.push(ValidationError::TooShort { 
            field: "name".into(), 
            min: 2 
        });
    }
    
    // ...
}
```

## 7. 配置选项

```rust
enum CacheDriver {
    Memory,
    Redis { host: String, port: u16 },
    Memcached { servers: Vec<String> },
    File { path: String },
}

enum DatabaseDriver {
    Postgres { url: String },
    MySQL { url: String },
    SQLite { path: String },
}
```

## Option 和 Result 也是 Enum

它们没有任何魔法，就是标准库定义的 Enum：

```rust
// Option 的真实定义
enum Option<T> {
    Some(T),
    None,
}

// Result 的真实定义
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

你完全可以自己造一个：

```rust
enum MyOption<T> {
    Something(T),
    Nothing,
}
```

## 核心价值

| 问题 | 用字符串/数字 | 用 Enum |
|-----|--------------|---------|
| Typo | 运行时才发现 | 编译期报错 |
| 漏处理情况 | 可能忘记 | match 强制穷尽 |
| 每种状态带不同数据 | 塞一堆 nullable 字段 | 每个变体精确定义 |
| 重构 | 全局搜索字符串 | 改 Enum，编译器告诉你哪里要改 |

**一句话**：凡是「一个东西可能是几种情况之一」的场景，都该用 Enum。编译器帮你兜底。

---

**下节课**：泛型 Generics
