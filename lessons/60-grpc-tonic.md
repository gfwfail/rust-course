# 第 60 课：gRPC 与 tonic

## 🎯 今日主题

gRPC 是 Google 开发的高性能 RPC 框架，在微服务架构中广泛使用。今天我们学习如何在 Rust 中使用 `tonic` 来构建 gRPC 服务。

---

## 📦 为什么用 gRPC？

对比 REST API：

| 特性 | REST | gRPC |
|------|------|------|
| 协议 | HTTP/1.1 JSON | HTTP/2 Protobuf |
| 性能 | 较慢（文本解析） | 快（二进制） |
| 类型安全 | 弱 | 强（.proto 定义） |
| 流式传输 | 需要 WebSocket | 原生支持 |
| 代码生成 | 可选 | 必须 |

**适用场景**：微服务内部通信、高性能 API、需要双向流的场景。

---

## 🛠️ 项目设置

```toml
# Cargo.toml
[package]
name = "grpc-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
tonic = "0.12"
prost = "0.13"           # Protobuf 运行时
tokio = { version = "1", features = ["full"] }

[build-dependencies]
tonic-build = "0.12"     # 编译时生成代码
```

---

## 📝 定义 Protobuf

创建 `proto/user.proto`：

```protobuf
syntax = "proto3";

package user;

// 用户服务
service UserService {
    // 一元 RPC：获取单个用户
    rpc GetUser(GetUserRequest) returns (User);
    
    // 一元 RPC：创建用户
    rpc CreateUser(CreateUserRequest) returns (User);
    
    // 服务端流：获取用户列表
    rpc ListUsers(ListUsersRequest) returns (stream User);
}

message GetUserRequest {
    int64 id = 1;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message ListUsersRequest {
    int32 page_size = 1;
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int64 created_at = 4;
}
```

---

## ⚙️ 构建脚本

创建 `build.rs`：

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 编译 proto 文件，生成 Rust 代码
    tonic_build::configure()
        .build_server(true)   // 生成服务端代码
        .build_client(true)   // 生成客户端代码
        .compile_protos(
            &["proto/user.proto"],  // proto 文件
            &["proto"],             // include 目录
        )?;
    Ok(())
}
```

---

## 🖥️ 实现服务端

```rust
// src/server.rs
use tonic::{transport::Server, Request, Response, Status};
use tokio_stream::wrappers::ReceiverStream;
use std::sync::Arc;
use tokio::sync::RwLock;

// 引入生成的代码
pub mod user {
    tonic::include_proto!("user");
}

use user::{
    user_service_server::{UserService, UserServiceServer},
    CreateUserRequest, GetUserRequest, ListUsersRequest, User,
};

// 服务实现
#[derive(Debug, Default)]
pub struct UserServiceImpl {
    users: Arc<RwLock<Vec<User>>>,
}

#[tonic::async_trait]
impl UserService for UserServiceImpl {
    // 获取用户
    async fn get_user(
        &self,
        request: Request<GetUserRequest>,
    ) -> Result<Response<User>, Status> {
        let req = request.into_inner();
        let users = self.users.read().await;
        
        users
            .iter()
            .find(|u| u.id == req.id)
            .cloned()
            .map(Response::new)
            .ok_or_else(|| Status::not_found("用户不存在"))
    }

    // 创建用户
    async fn create_user(
        &self,
        request: Request<CreateUserRequest>,
    ) -> Result<Response<User>, Status> {
        let req = request.into_inner();
        let mut users = self.users.write().await;
        
        let user = User {
            id: users.len() as i64 + 1,
            name: req.name,
            email: req.email,
            created_at: chrono::Utc::now().timestamp(),
        };
        
        users.push(user.clone());
        Ok(Response::new(user))
    }

    // 服务端流式返回
    type ListUsersStream = ReceiverStream<Result<User, Status>>;

    async fn list_users(
        &self,
        request: Request<ListUsersRequest>,
    ) -> Result<Response<Self::ListUsersStream>, Status> {
        let req = request.into_inner();
        let users = self.users.read().await.clone();
        
        let (tx, rx) = tokio::sync::mpsc::channel(4);
        
        tokio::spawn(async move {
            for user in users.into_iter().take(req.page_size as usize) {
                // 模拟延迟
                tokio::time::sleep(std::time::Duration::from_millis(100)).await;
                tx.send(Ok(user)).await.ok();
            }
        });

        Ok(Response::new(ReceiverStream::new(rx)))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse()?;
    let service = UserServiceImpl::default();

    println!("🚀 gRPC Server listening on {}", addr);

    Server::builder()
        .add_service(UserServiceServer::new(service))
        .serve(addr)
        .await?;

    Ok(())
}
```

---

## 📱 实现客户端

```rust
// src/client.rs
use user::user_service_client::UserServiceClient;
use user::{CreateUserRequest, GetUserRequest, ListUsersRequest};

pub mod user {
    tonic::include_proto!("user");
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 连接服务端
    let mut client = UserServiceClient::connect("http://[::1]:50051").await?;

    // 创建用户
    let response = client
        .create_user(CreateUserRequest {
            name: "张三".to_string(),
            email: "zhangsan@example.com".to_string(),
        })
        .await?;
    
    println!("✅ 创建用户: {:?}", response.into_inner());

    // 获取用户
    let response = client
        .get_user(GetUserRequest { id: 1 })
        .await?;
    
    println!("✅ 获取用户: {:?}", response.into_inner());

    // 流式获取用户列表
    let mut stream = client
        .list_users(ListUsersRequest { page_size: 10 })
        .await?
        .into_inner();

    println!("📋 用户列表（流式）:");
    while let Some(user) = stream.message().await? {
        println!("  - {:?}", user);
    }

    Ok(())
}
```

---

## 🔐 添加认证中间件

```rust
use tonic::{Request, Status};

// 认证拦截器
fn check_auth(req: Request<()>) -> Result<Request<()>, Status> {
    let token = req
        .metadata()
        .get("authorization")
        .and_then(|v| v.to_str().ok());

    match token {
        Some(t) if t.starts_with("Bearer ") => Ok(req),
        _ => Err(Status::unauthenticated("缺少有效 Token")),
    }
}

// 服务端使用拦截器
Server::builder()
    .add_service(UserServiceServer::with_interceptor(
        service,
        check_auth,
    ))
    .serve(addr)
    .await?;

// 客户端添加 Token
let mut client = UserServiceClient::with_interceptor(
    channel,
    |mut req: Request<()>| {
        req.metadata_mut().insert(
            "authorization",
            "Bearer my-token".parse().unwrap(),
        );
        Ok(req)
    },
);
```

---

## 🌊 双向流示例

```protobuf
// 在 proto 中添加
service ChatService {
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
    string user = 1;
    string content = 2;
}
```

```rust
// 服务端实现
type ChatStream = Pin<Box<dyn Stream<Item = Result<ChatMessage, Status>> + Send>>;

async fn chat(
    &self,
    request: Request<tonic::Streaming<ChatMessage>>,
) -> Result<Response<Self::ChatStream>, Status> {
    let mut stream = request.into_inner();
    
    let output = async_stream::try_stream! {
        while let Some(msg) = stream.message().await? {
            // Echo back with prefix
            yield ChatMessage {
                user: "Server".to_string(),
                content: format!("收到: {}", msg.content),
            };
        }
    };

    Ok(Response::new(Box::pin(output)))
}
```

---

## 💡 关键概念

### gRPC 四种模式

| 模式 | 请求 | 响应 | 场景 |
|------|------|------|------|
| 一元 | 单个 | 单个 | 普通 API |
| 服务端流 | 单个 | 流 | 分页/推送 |
| 客户端流 | 流 | 单个 | 上传/聚合 |
| 双向流 | 流 | 流 | 聊天/实时 |

### tonic 核心组件

- `tonic-build`: 编译时生成代码
- `prost`: Protobuf 序列化
- `#[tonic::async_trait]`: 异步 trait 支持
- `Request<T>` / `Response<T>`: 请求响应包装
- `Status`: gRPC 错误类型

---

## 🎯 最佳实践

1. **Proto 组织**：按服务拆分文件，使用 `import`
2. **错误处理**：使用 `Status` 的语义化错误码
3. **超时设置**：客户端设置合理的 deadline
4. **健康检查**：实现 `grpc.health.v1.Health` 服务
5. **连接池**：复用 `Channel`，不要每次新建

---

## 📚 参考资源

- [tonic 官方文档](https://docs.rs/tonic)
- [gRPC 官网](https://grpc.io/)
- [Protocol Buffers 文档](https://protobuf.dev/)

---

*课程日期：2026-02-20*
