# 第 55 课：WebSocket 实时通信

## 📚 本节内容

上节课我们学了 Redis，今天来讲 **WebSocket**——实时双向通信的基石。

### 为什么需要 WebSocket？

HTTP 是"请求-响应"模型：客户端问，服务器答。但有些场景需要服务器**主动推送**：

- 💬 聊天应用
- 📈 实时股票行情
- 🎮 在线游戏
- 🔔 通知系统

WebSocket 建立后，双方可以**随时发送消息**，不用等对方先说话。

---

## 🛠️ 依赖

```toml
[dependencies]
axum = { version = "0.8", features = ["ws"] }
tokio = { version = "1", features = ["full"] }
tokio-tungstenite = "0.26"
futures = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

---

## 🎯 基础示例：Echo 服务器

最简单的 WebSocket 服务——收到什么就回什么：

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    response::IntoResponse,
    routing::get,
    Router,
};
use futures::{SinkExt, StreamExt};

async fn ws_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    // 升级 HTTP 连接为 WebSocket
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    // 循环处理消息
    while let Some(Ok(msg)) = socket.next().await {
        match msg {
            Message::Text(text) => {
                println!("收到: {text}");
                // 原样返回
                if socket.send(Message::Text(text)).await.is_err() {
                    break; // 发送失败，连接断开
                }
            }
            Message::Close(_) => {
                println!("客户端断开");
                break;
            }
            _ => {} // 忽略 Binary/Ping/Pong
        }
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/ws", get(ws_handler));
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    println!("WebSocket 服务运行在 ws://localhost:3000/ws");
    axum::serve(listener, app).await.unwrap();
}
```

### 核心概念

1. **`WebSocketUpgrade`** - Axum 提供的升级器，把 HTTP 升级为 WebSocket
2. **`on_upgrade()`** - 升级成功后执行的回调
3. **`socket.next()`** - 接收下一条消息（Stream）
4. **`socket.send()`** - 发送消息（Sink）

---

## 📤 分离读写：双向独立通信

实际应用中，发送和接收往往要**同时进行**：

```rust
use futures::stream::SplitSink;
use tokio::sync::mpsc;

async fn handle_socket(socket: WebSocket) {
    // 分离读写
    let (mut sender, mut receiver) = socket.split();
    
    // 创建内部通道
    let (tx, mut rx) = mpsc::channel::<String>(32);
    
    // 任务1：读取客户端消息
    let tx_clone = tx.clone();
    let read_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            if let Message::Text(text) = msg {
                println!("收到: {text}");
                // 处理后发回（通过 channel）
                let response = format!("服务器收到: {text}");
                let _ = tx_clone.send(response).await;
            }
        }
    });
    
    // 任务2：发送消息给客户端
    let write_task = tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            if sender.send(Message::Text(msg)).await.is_err() {
                break;
            }
        }
    });
    
    // 等待任一任务结束
    tokio::select! {
        _ = read_task => {},
        _ = write_task => {},
    }
}
```

### 为什么要分离？

- **读** 和 **写** 可以在不同的 task 里独立运行
- 服务器可以**主动推送**，不用等客户端发消息
- 用 `mpsc::channel` 在 task 之间传递数据

---

## 🏠 聊天室：广播给所有连接

真正的聊天室需要**广播**——一个人说话，所有人都能听到：

```rust
use axum::extract::State;
use std::sync::Arc;
use tokio::sync::broadcast;

// 共享状态
struct AppState {
    // 广播通道，所有连接共享
    tx: broadcast::Sender<String>,
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, state))
}

async fn handle_socket(socket: WebSocket, state: Arc<AppState>) {
    let (mut sender, mut receiver) = socket.split();
    
    // 订阅广播
    let mut rx = state.tx.subscribe();
    
    // 任务1：接收广播，转发给此客户端
    let send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            if sender.send(Message::Text(msg)).await.is_err() {
                break;
            }
        }
    });
    
    // 任务2：接收客户端消息，广播给所有人
    let tx = state.tx.clone();
    let recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = receiver.next().await {
            // 广播给所有订阅者
            let _ = tx.send(text);
        }
    });
    
    tokio::select! {
        _ = send_task => {},
        _ = recv_task => {},
    }
}

#[tokio::main]
async fn main() {
    // 创建广播通道（容量 100）
    let (tx, _) = broadcast::channel(100);
    let state = Arc::new(AppState { tx });
    
    let app = Router::new()
        .route("/ws", get(ws_handler))
        .with_state(state);
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### `broadcast::channel` vs `mpsc::channel`

| 类型 | 特点 |
|------|------|
| `mpsc` | 多生产者，**单**消费者 |
| `broadcast` | 多生产者，**多**消费者（每条消息所有人都收到） |

聊天室用 `broadcast`，因为每条消息要发给**所有人**。

---

## 💡 实际应用技巧

### 1. 心跳检测

WebSocket 连接可能"假死"，用心跳保活：

```rust
use std::time::Duration;
use tokio::time::interval;

async fn handle_socket(mut socket: WebSocket) {
    let mut heartbeat = interval(Duration::from_secs(30));
    
    loop {
        tokio::select! {
            // 收到客户端消息
            msg = socket.next() => {
                match msg {
                    Some(Ok(Message::Text(text))) => {
                        // 处理消息
                    }
                    Some(Ok(Message::Pong(_))) => {
                        // 收到 Pong，连接正常
                    }
                    _ => break,
                }
            }
            // 定时发送 Ping
            _ = heartbeat.tick() => {
                if socket.send(Message::Ping(vec![])).await.is_err() {
                    break; // 发送失败，连接已断
                }
            }
        }
    }
}
```

### 2. 带认证的 WebSocket

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct WsQuery {
    token: String,
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    Query(query): Query<WsQuery>,
) -> impl IntoResponse {
    // 验证 token
    if !verify_token(&query.token) {
        return (axum::http::StatusCode::UNAUTHORIZED, "Invalid token")
            .into_response();
    }
    
    ws.on_upgrade(handle_socket).into_response()
}
```

客户端连接：`ws://localhost:3000/ws?token=xxx`

### 3. JSON 消息协议

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum WsMessage {
    #[serde(rename = "chat")]
    Chat { content: String },
    #[serde(rename = "join")]
    Join { room: String },
    #[serde(rename = "leave")]
    Leave { room: String },
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(Ok(Message::Text(text))) = socket.next().await {
        match serde_json::from_str::<WsMessage>(&text) {
            Ok(WsMessage::Chat { content }) => {
                println!("聊天: {content}");
            }
            Ok(WsMessage::Join { room }) => {
                println!("加入房间: {room}");
            }
            Ok(WsMessage::Leave { room }) => {
                println!("离开房间: {room}");
            }
            Err(e) => {
                println!("解析失败: {e}");
            }
        }
    }
}
```

---

## ⚠️ 常见坑

1. **忘记处理连接断开**
   ```rust
   // ❌ 错误：没检查 send 结果
   socket.send(msg).await;
   
   // ✅ 正确：检查并退出
   if socket.send(msg).await.is_err() {
       break;
   }
   ```

2. **阻塞 WebSocket 循环**
   ```rust
   // ❌ 错误：耗时操作阻塞循环
   while let Some(msg) = socket.next().await {
       do_heavy_work().await; // 太慢！
   }
   
   // ✅ 正确：spawn 新 task
   while let Some(msg) = socket.next().await {
       let msg = msg.clone();
       tokio::spawn(async move {
           do_heavy_work().await;
       });
   }
   ```

3. **广播通道满了**
   ```rust
   // broadcast 通道满了会丢弃旧消息
   // 用 send() 不会阻塞，但接收方可能 lag
   let _ = tx.send(msg); // 忽略错误
   ```

---

## 📝 小结

| 概念 | 说明 |
|------|------|
| `WebSocketUpgrade` | HTTP → WebSocket 升级 |
| `socket.split()` | 分离读写，并行处理 |
| `broadcast::channel` | 一对多广播 |
| `Ping/Pong` | 心跳检测 |

WebSocket 是实时应用的基石。Axum 的 WebSocket 支持简洁优雅，配合 Tokio 的异步特性，写实时服务很舒服。

---

下节课我们讲 **gRPC**——另一种高性能通信方式。
