# 🦀 Rust 学习小组课程笔记

> 授课时间：2026-02-14 ~ 2026-02-16  
> 授课对象：Web 程序员背景（Laravel/PHP/JS）  
> 教学风格：实战导向，对比其他语言，讲「为什么」而不只是「怎么写」

---

## 📚 课程目录

### 基础篇
| 课程 | 主题 | 链接 |
|------|------|------|
| 01 | Hello World 与基础概念 | [查看](lessons/01-hello-world.md) |
| 02 | 变量与数据类型 | [查看](lessons/02-variables.md) |
| 03 | 所有权 Ownership | [查看](lessons/03-ownership.md) |
| 04 | 生命周期 Lifetime | [查看](lessons/04-lifetime.md) |
| 05 | 错误处理 | [查看](lessons/05-error-handling.md) |
| 06 | Rust 设计模式 | [查看](lessons/06-design-patterns.md) |

### 语法与类型系统
| 课程 | 主题 | 链接 |
|------|------|------|
| 07 | Rust 特殊语法 | [查看](lessons/07-special-syntax.md) |
| 08 | Enum 实战场景 | [查看](lessons/08-enum-practical.md) |
| 09 | 泛型 Generics | [查看](lessons/09-generics.md) |
| 10 | Trait 入门 | [查看](lessons/10-trait-basics.md) |
| 11 | Trait 对象与 dyn | [查看](lessons/11-trait-objects.md) |

### 集合与迭代
| 课程 | 主题 | 链接 |
|------|------|------|
| 12 | Vec 动态数组 | [查看](lessons/12-vec.md) |
| 13 | String 深入 | [查看](lessons/13-string.md) |
| 14 | HashMap | [查看](lessons/14-hashmap.md) |
| 15 | 迭代器 Iterators | [查看](lessons/15-iterators.md) |
| 16 | 闭包 Closures | [查看](lessons/16-closures.md) |

### 智能指针
| 课程 | 主题 | 链接 |
|------|------|------|
| 17 | Box 堆分配 | [查看](lessons/17-box.md) |
| 18 | Rc 引用计数 | [查看](lessons/18-rc.md) |
| 19 | RefCell 内部可变性 | [查看](lessons/19-refcell.md) |
| 20 | Weak 与循环引用 | [查看](lessons/20-weak.md) |

### 并发编程
| 课程 | 主题 | 链接 |
|------|------|------|
| 21 | 线程基础 | [查看](lessons/21-threads.md) |
| 22 | Channel 消息传递 | [查看](lessons/22-channel.md) |
| 23 | Mutex 互斥锁 | [查看](lessons/23-mutex.md) |
| 24 | RwLock 读写锁 | [查看](lessons/24-rwlock.md) |
| 25 | Send 与 Sync | [查看](lessons/25-send-sync.md) |
| 26 | 原子类型 Atomics | [查看](lessons/26-atomics.md) |

### 异步编程
| 课程 | 主题 | 链接 |
|------|------|------|
| 27 | async/await 入门 | [查看](lessons/27-async-await.md) |
| 28 | Tokio 运行时 | [查看](lessons/28-tokio.md) |
| 29 | Axum Web 框架入门 | [查看](lessons/29-axum.md) |
| 30 | SQLx 数据库操作 | [查看](lessons/30-sqlx.md) |

### Web 开发进阶
| 课程 | 主题 | 链接 |
|------|------|------|
| 31 | 错误处理最佳实践 (anyhow/thiserror) | [查看](lessons/31-anyhow-thiserror.md) |
| 32 | Serde 序列化与反序列化 | [查看](lessons/32-serde.md) |
| 33 | Tracing 日志与追踪 | [查看](lessons/33-tracing.md) |
| 34 | 测试 (Testing) | [查看](lessons/34-testing.md) |

### 元编程
| 课程 | 主题 | 链接 |
|------|------|------|
| 35 | 宏 (Macros) 入门 | [查看](lessons/35-macros.md) |
| 36 | 过程宏 (Procedural Macros) | [查看](lessons/36-proc-macros.md) |

### 进阶篇
| 课程 | 主题 | 链接 |
|------|------|------|
| 37 | Unsafe Rust | [查看](lessons/37-unsafe.md) |
| 38 | FFI 与 C 语言交互 | [查看](lessons/38-ffi.md) |

### 工程化
| 课程 | 主题 | 链接 |
|------|------|------|
| 39 | Cargo Workspace 工作空间 | [查看](lessons/39-workspace.md) |
| 40 | Cargo Features 条件编译 | [查看](lessons/40-cargo-features.md) |

### 实战项目
| 课程 | 主题 | 链接 |
|------|------|------|
| 41 | CLI 工具开发 (clap) | [查看](lessons/41-clap.md) |
| 42 | 配置管理 (config crate) | [查看](lessons/42-config.md) |
| 43 | 环境变量管理 (dotenvy) | [查看](lessons/43-dotenvy.md) |
| 44 | HTTP 客户端 (reqwest) | [查看](lessons/44-reqwest.md) |
| 45 | 日期时间处理 (chrono) | [查看](lessons/45-chrono.md) |
| 46 | UUID 生成 (uuid) | [查看](lessons/46-uuid.md) |
| 47 | 正则表达式 (regex) | [查看](lessons/47-regex.md) |
| 48 | 全局状态与延迟初始化 (LazyLock) | [查看](lessons/48-lazylock.md) |
| 49 | 随机数生成 (rand) | [查看](lessons/49-rand.md) |
| 50 | 密码哈希 (argon2/bcrypt) | [查看](lessons/50-password-hashing.md) |
| 51 | JWT 认证 (jsonwebtoken) | [查看](lessons/51-jwt.md) |
| 52 | 数据验证 (validator) | [查看](lessons/52-validator.md) |
| 53 | Tower 中间件 (tower) | [查看](lessons/53-tower.md) |
| 54 | Redis 集成 (redis) | [查看](lessons/54-redis.md) |
| 55 | WebSocket 实时通信 | [查看](lessons/55-websocket.md) |
| 56 | 限流 Rate Limiting | [查看](lessons/56-rate-limiting.md) |
| 57 | 优雅关闭 Graceful Shutdown | [查看](lessons/57-graceful-shutdown.md) |
| 58 | OpenAPI 文档生成 (utoipa) | [查看](lessons/58-utoipa.md) |
| 59 | 后台任务与定时调度 | [查看](lessons/59-background-jobs.md) |
| 60 | gRPC 与 tonic | [查看](lessons/60-grpc-tonic.md) |
| 61 | Prometheus 监控指标 (metrics) | [查看](lessons/61-prometheus-metrics.md) |
| 62 | 内存缓存 (moka) | [查看](lessons/62-moka.md) |
| 63 | 文件上传与 Multipart 处理 | [查看](lessons/63-multipart-upload.md) |
| 64 | S3 对象存储 (aws-sdk-s3) | [查看](lessons/64-s3-storage.md) |

### 语言深入篇（回归本质）
| 课程 | 主题 | 链接 |
|------|------|------|
| 65 | 模式匹配深入 | [查看](lessons/65-pattern-matching.md) |
| 66 | 类型转换 (From/Into/TryFrom/AsRef) | [查看](lessons/66-type-conversions.md) |
| 67 | Deref 与智能指针的魔法 | [查看](lessons/67-deref-smart-pointers.md) |
| 68 | Drop trait 与资源管理 | [查看](lessons/68-drop-trait.md) |
| 69 | Rc 与 Arc — 共享所有权 | [查看](lessons/69-rc-arc.md) |
| 70 | RefCell 与内部可变性 | [查看](lessons/70-refcell-interior-mutability.md) |
| 71 | Cell — 轻量级内部可变性 | [查看](lessons/71-cell.md) |
| 72 | Cow — 写时克隆的智能指针 | [查看](lessons/72-cow.md) |
| 73 | PhantomData — 幽灵数据与零大小类型 | [查看](lessons/73-phantomdata.md) |
| 74 | Pin 与 Unpin — 固定内存位置 | [查看](lessons/74-pin-unpin.md) |
| 75 | ManuallyDrop — 手动控制析构 | [查看](lessons/75-manually-drop.md) |
| 76 | MaybeUninit — 未初始化内存的安全处理 | [查看](lessons/76-maybeuninit.md) |
| 77 | NonNull — 非空裸指针的安全封装 | [查看](lessons/77-nonnull.md) |
| 78 | UnsafeCell — 内部可变性的基石 | [查看](lessons/78-unsafecell.md) |
| 79 | std::mem — 内存操作工具箱 | [查看](lessons/79-std-mem.md) |
| 80 | std::ops — 操作符重载 | [查看](lessons/80-std-ops.md) |
| 81 | std::cmp — 比较与排序 | [查看](lessons/81-std-cmp.md) |
| 82 | std::hash — 哈希的艺术 | [查看](lessons/82-std-hash.md) |
| 83 | std::fmt — 格式化输出 | [查看](lessons/83-std-fmt.md) |
| 84 | std::iter — 迭代器的力量 | [查看](lessons/84-std-iter.md) |
| 85 | std::borrow — Borrow 和 ToOwned | [查看](lessons/85-std-borrow.md) |
| 86 | std::default — Default trait 与默认值 | [查看](lessons/86-std-default.md) |
| 87 | std::clone — 克隆与复制的艺术 | [查看](lessons/87-std-clone.md) |
| 88 | std::marker — Marker Traits 的奥秘 | [查看](lessons/88-std-marker.md) |
| 89 | std::any — 运行时类型信息 (RTTI) | [查看](lessons/89-std-any.md) |
| 90 | std::io — 输入输出的艺术 | [查看](lessons/90-std-io.md) |
| 91 | std::fs — 文件系统操作 | [查看](lessons/91-std-fs.md) |
| 92 | std::path — 路径操作的艺术 | [查看](lessons/92-std-path.md) |
| 93 | std::env — 环境变量与命令行参数 | [查看](lessons/93-std-env.md) |
| 94 | std::process — 进程控制与子进程 | [查看](lessons/94-std-process.md) |
| 95 | std::net — 网络编程基础 | [查看](lessons/95-std-net.md) |
| 96 | std::time — 时间与持续时间 | [查看](lessons/96-std-time.md) |
| 97 | std::thread — 多线程编程基础 | [查看](lessons/97-std-thread.md) |
| 98 | std::sync — 同步原语 | [查看](lessons/98-std-sync.md) |
| 99 | std::collections — 标准库集合类型全览 | [查看](lessons/99-std-collections.md) |
| 100 | 🎉 模式匹配进阶 — Rust 最强大的武器 | [查看](lessons/100-pattern-matching-advanced.md) |
| 101 | std::error — Error trait 与错误链 | [查看](lessons/101-std-error.md) |

---

## 🎯 学习路线

```
基础语法 → 所有权/借用/生命周期 → 错误处理
    ↓
泛型与 Trait → 集合与迭代器 → 闭包
    ↓
智能指针 → 并发编程 → 异步编程
    ↓
Web 框架实战（Axum/Actix）
```

---

## 💡 核心概念速记

### 所有权三铁律
1. 每个值有且只有一个 owner
2. owner 离开作用域，值自动销毁
3. 值可以 move 或 borrow

### 借用规则
- 多个 `&T` 或一个 `&mut T`
- 不能同时存在

### 错误处理
- `Option<T>` = 有或没有
- `Result<T, E>` = 成功或失败
- `?` = 错误传播

### 并发安全
- `Send` = 可跨线程移动
- `Sync` = 可跨线程共享
- `Arc<Mutex<T>>` = 多线程共享可变

---

## 📖 推荐资源

- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Rustlings](https://github.com/rust-lang/rustlings)

---

*笔记整理：性奴001*  
*最后更新：2026-02-27 06:00*
