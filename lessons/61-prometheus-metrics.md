# 第 61 课：Prometheus 监控指标 (metrics)

> 授课时间：2026-02-20  
> 主题：使用 metrics crate 实现 Prometheus 监控

---

## 📌 为什么需要监控指标？

之前我们学了 `tracing` 做日志追踪，但日志是**定性**的（发生了什么），而指标是**定量**的（有多少、多快、多大）。

生产环境必须两者结合：
- **日志**：调试问题，追踪请求链路
- **指标**：监控健康状态，设置告警，发现趋势

常见指标：
- 请求量（QPS）
- 延迟分布（P50/P95/P99）
- 错误率
- 内存/CPU 使用
- 数据库连接池状态

---

## 🔧 核心 Crate

```toml
[dependencies]
metrics = "0.24"              # 指标接口（类似 tracing）
metrics-exporter-prometheus = "0.16"  # Prometheus 导出器
axum = "0.8"
tokio = { version = "1", features = ["full"] }
```

Rust 的 `metrics` 生态跟 `tracing` 类似：
- `metrics` = 接口层（定义指标）
- `metrics-exporter-*` = 后端（Prometheus/StatsD/etc）

---

## 📝 基础用法

```rust
use metrics::{counter, gauge, histogram};

// Counter: 只增不减（请求数、错误数）
counter!("http_requests_total", 
    "method" => "GET", 
    "path" => "/api/users"
).increment(1);

// Gauge: 可增可减（当前连接数、队列长度）
gauge!("db_connections_active").set(42.0);

// Histogram: 分布统计（延迟、大小）
histogram!("http_request_duration_seconds").record(0.025);
```

### 三种指标类型

| 类型 | 特点 | 典型用途 |
|------|------|----------|
| `counter` | 只增不减，重启归零 | 请求总数、错误总数 |
| `gauge` | 当前值，可增可减 | 内存使用、队列长度 |
| `histogram` | 分布统计，自动计算分位数 | 延迟、请求大小 |

---

## 🚀 Axum 集成示例

```rust
use axum::{routing::get, Router};
use metrics::{counter, histogram};
use metrics_exporter_prometheus::{Matcher, PrometheusBuilder, PrometheusHandle};
use std::time::Instant;

// 初始化 Prometheus exporter
fn setup_metrics() -> PrometheusHandle {
    PrometheusBuilder::new()
        // 自定义 histogram bucket（延迟分布）
        .set_buckets_for_metric(
            Matcher::Full("http_request_duration_seconds".to_string()),
            &[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
        )
        .unwrap()
        .install_recorder()
        .unwrap()
}

async fn handler() -> &'static str {
    let start = Instant::now();
    
    // 业务逻辑...
    
    // 记录指标
    histogram!("handler_duration_seconds")
        .record(start.elapsed().as_secs_f64());
    counter!("handler_calls_total").increment(1);
    
    "Hello!"
}

#[tokio::main]
async fn main() {
    let handle = setup_metrics();
    
    let app = Router::new()
        .route("/", get(handler))
        // 暴露 /metrics 端点给 Prometheus 抓取
        .route("/metrics", get(move || {
            let h = handle.clone();
            async move { h.render() }
        }));
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

## 📊 /metrics 端点输出

访问 `http://localhost:3000/metrics`：

```prometheus
# TYPE handler_calls_total counter
handler_calls_total 42

# TYPE handler_duration_seconds histogram
handler_duration_seconds_bucket{le="0.005"} 30
handler_duration_seconds_bucket{le="0.01"} 38
handler_duration_seconds_bucket{le="0.025"} 40
handler_duration_seconds_bucket{le="+Inf"} 42
handler_duration_seconds_sum 0.325
handler_duration_seconds_count 42
```

Prometheus 定期抓取这个端点 → Grafana 可视化展示

---

## 💡 实用技巧

### 1. 业务指标

```rust
// 订单相关
counter!("orders_created_total", "status" => "success").increment(1);
counter!("orders_created_total", "status" => "failed").increment(1);

// 支付金额
histogram!("payment_amount_usd").record(99.99);

// 库存
gauge!("inventory_items", "product" => "widget").set(1500.0);
```

### 2. 数据库连接池监控

```rust
use sqlx::PgPool;

async fn record_pool_metrics(pool: &PgPool) {
    let size = pool.size() as f64;
    let idle = pool.num_idle() as f64;
    
    gauge!("db_pool_connections_total").set(size);
    gauge!("db_pool_connections_idle").set(idle);
    gauge!("db_pool_connections_active").set(size - idle);
}
```

### 3. 自定义 Histogram Bucket

```rust
use metrics_exporter_prometheus::Matcher;

PrometheusBuilder::new()
    .set_buckets_for_metric(
        Matcher::Full("http_request_duration_seconds".to_string()),
        &[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
    )
    .unwrap()
    .install_recorder()
    .unwrap();
```

### 4. 添加指标描述

```rust
use metrics::describe_counter;

fn describe_metrics() {
    describe_counter!(
        "http_requests_total",
        "Total number of HTTP requests received"
    );
}
```

---

## ⚠️ 注意事项

### 1. 高基数问题（最常见的坑！）

```rust
// ❌ 错误：user_id 无限多，指标会爆炸
counter!("requests", "user_id" => user_id);

// ✅ 正确：用有限的分类
counter!("requests", "user_type" => "premium");
counter!("requests", "region" => "asia");
```

Label 的每个唯一组合都会创建一个新的时间序列。如果 label 值是无限的（如 user_id、request_id），会导致：
- 内存爆炸
- Prometheus 查询变慢
- 存储成本飙升

### 2. 命名规范

- 单位后缀：`_seconds`、`_bytes`、`_total`
- 蛇形命名：`http_request_duration_seconds`
- 避免动词：`requests_total` ✅ vs `count_requests` ❌

### 3. Histogram vs Summary

- **Histogram**：在服务端聚合（推荐），可以跨实例计算分位数
- **Summary**：在客户端计算分位数，不能聚合

---

## 🎯 生产环境架构

```
[Your App] ---> [Prometheus] ---> [Grafana]
   :3000/metrics    :9090           :3000
```

### Prometheus 配置 (`prometheus.yml`)

```yaml
scrape_configs:
  - job_name: 'my-rust-app'
    static_configs:
      - targets: ['localhost:3000']
    metrics_path: '/metrics'
    scrape_interval: 15s
```

---

## 🔗 与 tracing 配合

```rust
use tracing::info;
use metrics::counter;

async fn process_order(order_id: &str) -> Result<(), Error> {
    // 日志：详细追踪
    info!(order_id, "Processing order");
    
    // 指标：聚合统计
    counter!("orders_processed_total").increment(1);
    
    // ...
}
```

**日志回答「发生了什么」，指标回答「有多少」**

---

## 📚 相关资源

- [metrics crate 文档](https://docs.rs/metrics)
- [metrics-exporter-prometheus 文档](https://docs.rs/metrics-exporter-prometheus)
- [Prometheus 官方文档](https://prometheus.io/docs/)
- [Grafana 官方文档](https://grafana.com/docs/)

---

## 🎯 作业

1. 给你的 Axum API 添加 `/metrics` 端点
2. 记录：请求数、延迟分布、错误率
3. 用 Prometheus + Grafana 可视化

---

*下节课预告：OpenTelemetry 统一观测*
