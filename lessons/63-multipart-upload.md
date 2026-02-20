# 第 63 课：文件上传与 Multipart 处理

> 日期：2026-02-20  
> 主题：在 Axum 中处理文件上传

---

## 🎯 为什么要单独讲这个？

文件上传看似简单，但涉及很多细节：
- 内存限制（大文件不能全部加载到内存）
- 流式处理（边接收边写入）
- 文件类型验证
- 安全性（防止恶意文件）

---

## 📦 依赖配置

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4"] }
serde = { version = "1", features = ["derive"] }
```

---

## 📝 基础文件上传

```rust
use axum::{
    Router, routing::post,
    extract::Multipart,
    response::Json,
};
use serde::Serialize;
use std::path::Path;
use uuid::Uuid;

#[derive(Serialize)]
struct UploadResponse {
    filename: String,
    size: u64,
    content_type: Option<String>,
}

async fn upload_file(
    mut multipart: Multipart,
) -> Result<Json<Vec<UploadResponse>>, (axum::http::StatusCode, String)> {
    let mut results = Vec::new();
    let upload_dir = Path::new("./uploads");
    
    // 确保上传目录存在
    tokio::fs::create_dir_all(upload_dir).await
        .map_err(|e| (axum::http::StatusCode::INTERNAL_SERVER_ERROR, 
            format!("Failed to create upload dir: {}", e)))?;

    // 遍历所有 field
    while let Some(field) = multipart.next_field().await
        .map_err(|e| (axum::http::StatusCode::BAD_REQUEST, 
            format!("Multipart error: {}", e)))?
    {
        let original_name = field.file_name()
            .map(|s| s.to_string())
            .unwrap_or_else(|| "unknown".to_string());
        
        let content_type = field.content_type()
            .map(|s| s.to_string());

        // 生成唯一文件名（防止覆盖和路径遍历攻击）
        let extension = Path::new(&original_name)
            .extension()
            .and_then(|e| e.to_str())
            .unwrap_or("bin");
        let safe_filename = format!("{}.{}", Uuid::new_v4(), extension);
        let file_path = upload_dir.join(&safe_filename);

        // 读取数据并写入文件
        let data = field.bytes().await.map_err(|e| 
            (axum::http::StatusCode::BAD_REQUEST, 
             format!("Failed to read field: {}", e)))?;

        let size = data.len() as u64;
        tokio::fs::write(&file_path, &data).await
            .map_err(|e| (axum::http::StatusCode::INTERNAL_SERVER_ERROR, 
                format!("Failed to write file: {}", e)))?;

        results.push(UploadResponse {
            filename: safe_filename,
            size,
            content_type,
        });
    }

    Ok(Json(results))
}
```

---

## 🌊 流式上传（大文件）

上面的方式会把整个文件加载到内存。对于大文件，要用流式处理：

```rust
use tokio::fs::File;
use tokio::io::{AsyncWriteExt, BufWriter};

async fn upload_large_file(
    mut multipart: Multipart,
) -> Result<Json<UploadResponse>, (axum::http::StatusCode, String)> {
    let upload_dir = Path::new("./uploads");
    tokio::fs::create_dir_all(upload_dir).await.ok();

    while let Some(mut field) = multipart.next_field().await
        .map_err(|e| (axum::http::StatusCode::BAD_REQUEST, e.to_string()))?
    {
        let original_name = field.file_name()
            .map(|s| s.to_string())
            .unwrap_or_else(|| "unknown".to_string());
        
        let content_type = field.content_type().map(|s| s.to_string());
        
        let extension = Path::new(&original_name)
            .extension()
            .and_then(|e| e.to_str())
            .unwrap_or("bin");
        let safe_filename = format!("{}.{}", Uuid::new_v4(), extension);
        let file_path = upload_dir.join(&safe_filename);

        // 创建文件，使用 BufWriter 提高写入性能
        let file = File::create(&file_path).await
            .map_err(|e| (axum::http::StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
        let mut writer = BufWriter::new(file);
        let mut size: u64 = 0;

        // 流式读取并写入
        while let Some(chunk) = field.chunk().await
            .map_err(|e| (axum::http::StatusCode::BAD_REQUEST, e.to_string()))?
        {
            size += chunk.len() as u64;
            writer.write_all(&chunk).await
                .map_err(|e| (axum::http::StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
        }

        // 确保所有数据写入磁盘
        writer.flush().await.map_err(|e| 
            (axum::http::StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

        return Ok(Json(UploadResponse {
            filename: safe_filename,
            size,
            content_type,
        }));
    }

    Err((axum::http::StatusCode::BAD_REQUEST, "No file uploaded".to_string()))
}
```

---

## 🛡️ 文件验证

```rust
const MAX_FILE_SIZE: u64 = 10 * 1024 * 1024; // 10MB
const ALLOWED_TYPES: &[&str] = &[
    "image/jpeg", "image/png", "image/gif", "application/pdf"
];

fn validate_file(content_type: Option<&str>, size: u64) -> Result<(), String> {
    // 检查文件大小
    if size > MAX_FILE_SIZE {
        return Err(format!(
            "File too large: {} bytes (max: {} bytes)", 
            size, MAX_FILE_SIZE
        ));
    }

    // 检查文件类型
    if let Some(ct) = content_type {
        if !ALLOWED_TYPES.contains(&ct) {
            return Err(format!("File type not allowed: {}", ct));
        }
    } else {
        return Err("Missing content type".to_string());
    }

    Ok(())
}

// 更安全的方式：检查文件魔数（magic bytes）
fn validate_image_magic_bytes(data: &[u8]) -> bool {
    // JPEG: FF D8 FF
    if data.len() >= 3 && data[0..3] == [0xFF, 0xD8, 0xFF] {
        return true;
    }
    // PNG: 89 50 4E 47
    if data.len() >= 4 && data[0..4] == [0x89, 0x50, 0x4E, 0x47] {
        return true;
    }
    // GIF: 47 49 46 38
    if data.len() >= 4 && data[0..4] == [0x47, 0x49, 0x46, 0x38] {
        return true;
    }
    false
}
```

⚠️ **重要**：不要只信任 `content-type`，它可以被伪造。检查文件魔数更安全！

---

## 🔧 带文本字段的混合表单

实际场景中，表单往往同时包含文件和普通字段：

```rust
use std::collections::HashMap;

#[derive(Debug, Serialize)]
struct FormData {
    fields: HashMap<String, String>,
    files: Vec<UploadResponse>,
}

async fn upload_with_fields(
    mut multipart: Multipart,
) -> Result<Json<FormData>, (axum::http::StatusCode, String)> {
    let mut fields = HashMap::new();
    let mut files = Vec::new();
    let upload_dir = Path::new("./uploads");

    while let Some(field) = multipart.next_field().await
        .map_err(|e| (axum::http::StatusCode::BAD_REQUEST, e.to_string()))?
    {
        let name = field.name().map(|s| s.to_string()).unwrap_or_default();

        // 判断是文件还是普通字段
        if field.file_name().is_some() {
            // 这是文件
            let original_name = field.file_name().unwrap().to_string();
            let content_type = field.content_type().map(|s| s.to_string());
            
            let extension = Path::new(&original_name)
                .extension()
                .and_then(|e| e.to_str())
                .unwrap_or("bin");
            let safe_filename = format!("{}.{}", Uuid::new_v4(), extension);
            
            tokio::fs::create_dir_all(upload_dir).await.ok();
            let file_path = upload_dir.join(&safe_filename);
            
            let data = field.bytes().await.map_err(|e| 
                (axum::http::StatusCode::BAD_REQUEST, e.to_string()))?;
            
            tokio::fs::write(&file_path, &data).await.map_err(|e| 
                (axum::http::StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

            files.push(UploadResponse {
                filename: safe_filename,
                size: data.len() as u64,
                content_type,
            });
        } else {
            // 这是普通文本字段
            let value = field.text().await.map_err(|e| 
                (axum::http::StatusCode::BAD_REQUEST, e.to_string()))?;
            fields.insert(name, value);
        }
    }

    Ok(Json(FormData { fields, files }))
}
```

---

## 🚀 完整示例

```rust
#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/upload", post(upload_file))
        .route("/upload/large", post(upload_large_file))
        .route("/upload/form", post(upload_with_fields));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
        
    println!("Server running on http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

**测试命令：**
```bash
# 单文件上传
curl -X POST http://localhost:3000/upload \
  -F "file=@/path/to/image.jpg"

# 带字段的表单
curl -X POST http://localhost:3000/upload/form \
  -F "title=My Photo" \
  -F "description=A nice picture" \
  -F "file=@/path/to/image.jpg"
```

---

## 💡 最佳实践

1. **永远不要信任 `file_name`** - 用 UUID 生成新文件名
2. **验证 content-type** - 同时检查文件头（magic bytes）
3. **限制文件大小** - 在应用层和 nginx/代理层都设置
4. **流式处理大文件** - 避免内存溢出
5. **异步写入** - 使用 `BufWriter` 提高性能
6. **扫描病毒** - 生产环境考虑集成 ClamAV

---

## 📚 扩展阅读

- [axum Multipart 文档](https://docs.rs/axum/latest/axum/extract/struct.Multipart.html)
- [tokio fs 模块](https://docs.rs/tokio/latest/tokio/fs/index.html)

---

*下节课预告：S3 对象存储（aws-sdk-s3）*
