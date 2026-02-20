# 第 64 课：S3 对象存储 (aws-sdk-s3)

> 日期：2026-02-20  
> 主题：使用 aws-sdk-s3 操作云存储

---

## 🎯 为什么要学这个？

上节课讲了本地文件上传，但生产环境几乎都用云存储：
- **可扩展** - 无限存储空间
- **高可用** - 自动备份、CDN 加速
- **便宜** - 按需付费
- **兼容** - S3 协议是事实标准（Cloudflare R2、MinIO、阿里 OSS 都兼容）

---

## 📦 依赖配置

```toml
[dependencies]
aws-config = { version = "1", features = ["behavior-version-latest"] }
aws-sdk-s3 = "1"
aws-credential-types = "1"
tokio = { version = "1", features = ["full"] }
```

---

## 🔧 基础配置

```rust
use aws_config::BehaviorVersion;
use aws_sdk_s3::Client;

// 方式1：使用环境变量（推荐）
// 设置 AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION

async fn create_client() -> Client {
    let config = aws_config::defaults(BehaviorVersion::latest())
        .load()
        .await;
    Client::new(&config)
}

// 方式2：自定义配置（适用于 R2、MinIO 等）
use aws_credential_types::Credentials;

async fn create_r2_client() -> Client {
    let credentials = Credentials::new(
        "access_key_id",      // R2 Access Key ID
        "secret_access_key",  // R2 Secret Access Key
        None,
        None,
        "r2",
    );

    let config = aws_config::defaults(BehaviorVersion::latest())
        .credentials_provider(credentials)
        .endpoint_url("https://xxx.r2.cloudflarestorage.com")
        .region(aws_config::Region::new("auto"))
        .load()
        .await;
    
    Client::new(&config)
}
```

---

## 📤 上传文件

```rust
use aws_sdk_s3::primitives::ByteStream;
use std::path::Path;

async fn upload_file(
    client: &Client,
    bucket: &str,
    key: &str,          // S3 中的路径，如 "images/photo.jpg"
    file_path: &Path,
    content_type: Option<&str>,
) -> Result<String, aws_sdk_s3::Error> {
    let body = ByteStream::from_path(file_path)
        .await
        .expect("Failed to read file");

    let mut request = client
        .put_object()
        .bucket(bucket)
        .key(key)
        .body(body);

    if let Some(ct) = content_type {
        request = request.content_type(ct);
    }

    request.send().await?;
    
    // 返回文件 URL
    Ok(format!("https://{}.s3.amazonaws.com/{}", bucket, key))
}

// 从内存数据上传
async fn upload_bytes(
    client: &Client,
    bucket: &str,
    key: &str,
    data: Vec<u8>,
    content_type: &str,
) -> Result<(), aws_sdk_s3::Error> {
    client
        .put_object()
        .bucket(bucket)
        .key(key)
        .body(ByteStream::from(data))
        .content_type(content_type)
        .send()
        .await?;
    Ok(())
}
```

---

## 📥 下载文件

```rust
async fn download_file(
    client: &Client,
    bucket: &str,
    key: &str,
) -> Result<Vec<u8>, aws_sdk_s3::Error> {
    let response = client
        .get_object()
        .bucket(bucket)
        .key(key)
        .send()
        .await?;

    let data = response.body.collect().await
        .expect("Failed to read body")
        .into_bytes()
        .to_vec();

    Ok(data)
}

// 流式下载（大文件）
use tokio::io::AsyncWriteExt;
use tokio::fs::File;

async fn download_to_file(
    client: &Client,
    bucket: &str,
    key: &str,
    dest: &Path,
) -> Result<(), Box<dyn std::error::Error>> {
    let mut response = client
        .get_object()
        .bucket(bucket)
        .key(key)
        .send()
        .await?;

    let mut file = File::create(dest).await?;

    while let Some(chunk) = response.body.try_next().await? {
        file.write_all(&chunk).await?;
    }
    file.flush().await?;

    Ok(())
}
```

---

## 🔗 预签名 URL（重点！）

预签名 URL 让用户无需凭证就能临时访问/上传文件：

```rust
use aws_sdk_s3::presigning::PresigningConfig;
use std::time::Duration;

// 生成下载链接（GET）
async fn generate_download_url(
    client: &Client,
    bucket: &str,
    key: &str,
    expires_in_secs: u64,
) -> Result<String, Box<dyn std::error::Error>> {
    let presigning = PresigningConfig::expires_in(
        Duration::from_secs(expires_in_secs)
    )?;

    let presigned = client
        .get_object()
        .bucket(bucket)
        .key(key)
        .presigned(presigning)
        .await?;

    Ok(presigned.uri().to_string())
}

// 生成上传链接（PUT）- 让前端直传 S3
async fn generate_upload_url(
    client: &Client,
    bucket: &str,
    key: &str,
    content_type: &str,
    expires_in_secs: u64,
) -> Result<String, Box<dyn std::error::Error>> {
    let presigning = PresigningConfig::expires_in(
        Duration::from_secs(expires_in_secs)
    )?;

    let presigned = client
        .put_object()
        .bucket(bucket)
        .key(key)
        .content_type(content_type)
        .presigned(presigning)
        .await?;

    Ok(presigned.uri().to_string())
}
```

**前端使用预签名 URL 直传：**
```javascript
// 获取上传 URL
const { url } = await fetch('/api/upload-url').then(r => r.json());

// 直接上传到 S3
await fetch(url, {
    method: 'PUT',
    body: file,
    headers: { 'Content-Type': file.type }
});
```

---

## 📋 列出文件

```rust
async fn list_files(
    client: &Client,
    bucket: &str,
    prefix: Option<&str>,  // 文件夹前缀，如 "images/"
) -> Result<Vec<String>, aws_sdk_s3::Error> {
    let mut request = client.list_objects_v2().bucket(bucket);
    
    if let Some(p) = prefix {
        request = request.prefix(p);
    }

    let response = request.send().await?;
    
    let keys: Vec<String> = response
        .contents()
        .iter()
        .filter_map(|obj| obj.key().map(String::from))
        .collect();

    Ok(keys)
}

// 分页列出（大量文件）
async fn list_all_files(
    client: &Client,
    bucket: &str,
) -> Result<Vec<String>, aws_sdk_s3::Error> {
    let mut keys = Vec::new();
    let mut continuation_token: Option<String> = None;

    loop {
        let mut request = client.list_objects_v2().bucket(bucket);
        
        if let Some(token) = &continuation_token {
            request = request.continuation_token(token);
        }

        let response = request.send().await?;
        
        for obj in response.contents() {
            if let Some(key) = obj.key() {
                keys.push(key.to_string());
            }
        }

        if response.is_truncated() == Some(true) {
            continuation_token = response.next_continuation_token().map(String::from);
        } else {
            break;
        }
    }

    Ok(keys)
}
```

---

## 🗑️ 删除文件

```rust
// 删除单个文件
async fn delete_file(
    client: &Client,
    bucket: &str,
    key: &str,
) -> Result<(), aws_sdk_s3::Error> {
    client
        .delete_object()
        .bucket(bucket)
        .key(key)
        .send()
        .await?;
    Ok(())
}

// 批量删除
use aws_sdk_s3::types::{Delete, ObjectIdentifier};

async fn delete_files(
    client: &Client,
    bucket: &str,
    keys: Vec<&str>,
) -> Result<(), aws_sdk_s3::Error> {
    let objects: Vec<ObjectIdentifier> = keys
        .into_iter()
        .map(|k| ObjectIdentifier::builder().key(k).build().unwrap())
        .collect();

    client
        .delete_objects()
        .bucket(bucket)
        .delete(
            Delete::builder()
                .set_objects(Some(objects))
                .build()
                .unwrap()
        )
        .send()
        .await?;

    Ok(())
}
```

---

## 🚀 与 Axum 结合的完整示例

```rust
use axum::{
    Router, routing::{get, post},
    extract::{State, Path, Multipart},
    response::Json,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use uuid::Uuid;

struct AppState {
    s3: Client,
    bucket: String,
}

#[derive(Serialize)]
struct UploadResult {
    key: String,
    url: String,
}

// 上传文件
async fn upload(
    State(state): State<Arc<AppState>>,
    mut multipart: Multipart,
) -> Result<Json<UploadResult>, (axum::http::StatusCode, String)> {
    while let Some(field) = multipart.next_field().await
        .map_err(|e| (axum::http::StatusCode::BAD_REQUEST, e.to_string()))?
    {
        let filename = field.file_name()
            .map(|s| s.to_string())
            .unwrap_or_else(|| "unknown".to_string());
        
        let content_type = field.content_type()
            .map(|s| s.to_string())
            .unwrap_or_else(|| "application/octet-stream".to_string());

        // 生成唯一 key
        let ext = std::path::Path::new(&filename)
            .extension()
            .and_then(|e| e.to_str())
            .unwrap_or("bin");
        let key = format!("uploads/{}.{}", Uuid::new_v4(), ext);

        let data = field.bytes().await
            .map_err(|e| (axum::http::StatusCode::BAD_REQUEST, e.to_string()))?;

        // 上传到 S3
        state.s3
            .put_object()
            .bucket(&state.bucket)
            .key(&key)
            .body(ByteStream::from(data.to_vec()))
            .content_type(&content_type)
            .send()
            .await
            .map_err(|e| (axum::http::StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

        // 生成预签名 URL
        let presigning = PresigningConfig::expires_in(Duration::from_secs(3600)).unwrap();
        let url = state.s3
            .get_object()
            .bucket(&state.bucket)
            .key(&key)
            .presigned(presigning)
            .await
            .map_err(|e| (axum::http::StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?
            .uri()
            .to_string();

        return Ok(Json(UploadResult { key, url }));
    }

    Err((axum::http::StatusCode::BAD_REQUEST, "No file".to_string()))
}

// 获取预签名下载链接
async fn get_download_url(
    State(state): State<Arc<AppState>>,
    Path(key): Path<String>,
) -> Result<Json<serde_json::Value>, (axum::http::StatusCode, String)> {
    let presigning = PresigningConfig::expires_in(Duration::from_secs(3600)).unwrap();
    
    let url = state.s3
        .get_object()
        .bucket(&state.bucket)
        .key(&key)
        .presigned(presigning)
        .await
        .map_err(|e| (axum::http::StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?
        .uri()
        .to_string();

    Ok(Json(serde_json::json!({ "url": url })))
}

#[tokio::main]
async fn main() {
    let config = aws_config::defaults(BehaviorVersion::latest())
        .load()
        .await;
    
    let state = Arc::new(AppState {
        s3: Client::new(&config),
        bucket: std::env::var("S3_BUCKET").unwrap_or_else(|_| "my-bucket".into()),
    });

    let app = Router::new()
        .route("/upload", post(upload))
        .route("/files/:key", get(get_download_url))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

## 💡 最佳实践

1. **使用预签名 URL 直传** - 减轻后端负担，大文件直接传到 S3
2. **设置合理的过期时间** - 下载链接 1-24 小时，上传链接 5-15 分钟
3. **用 CDN 加速** - CloudFront / Cloudflare 放在 S3 前面
4. **设置 CORS** - S3 bucket 要配置允许前端域名
5. **生命周期策略** - 自动清理过期文件，节省成本

---

## 🔄 对比其他服务

| 服务 | endpoint_url | 区别 |
|------|-------------|------|
| AWS S3 | 默认 | 标准 |
| Cloudflare R2 | `https://xxx.r2.cloudflarestorage.com` | 无出口费用 |
| MinIO | `http://localhost:9000` | 自托管 |
| 阿里 OSS | `https://oss-cn-xxx.aliyuncs.com` | 需要特殊签名 |

---

## 📚 扩展阅读

- [aws-sdk-s3 文档](https://docs.rs/aws-sdk-s3/latest/)
- [S3 预签名请求](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)

---

*下节课预告：Sea-ORM 数据库 ORM*
