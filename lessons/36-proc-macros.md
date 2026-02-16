# 第 36 课：过程宏 (Procedural Macros)

> 📅 2026-02-17 06:00 (AEDT)

---

## 上节回顾

上节课学了 `macro_rules!` 声明式宏——通过模式匹配替换代码。

今天学 **过程宏**，它更强大：直接操作 Rust 的抽象语法树 (AST)！

---

## 三种过程宏

```rust
// 1. 派生宏 - 最常见
#[derive(Debug, Clone, Serialize)]
struct User { ... }

// 2. 属性宏 - 修饰函数/结构体
#[route("GET", "/users")]
async fn get_users() { ... }

// 3. 函数式过程宏 - 看起来像函数调用
let sql = sql!(SELECT * FROM users WHERE id = ?);
```

---

## 为什么过程宏需要单独 crate？

**过程宏在编译期运行**，它们是编译器的插件。

Rust 要求过程宏必须放在单独的 crate，设置 `proc-macro = true`：

```toml
# my_macros/Cargo.toml
[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }  # 解析 Rust 代码
quote = "1"                                     # 生成 Rust 代码
proc-macro2 = "1"                               # Token 操作
```

**常用工具链：**
- `syn` — 把 TokenStream 解析成 AST
- `quote` — 把 Rust 代码转回 TokenStream
- `proc-macro2` — 更好的 token 处理

---

## 派生宏 (Derive Macro)

最常见的过程宏！给结构体自动实现 trait。

**目标：** 实现一个 `#[derive(HelloMacro)]`，自动生成 `hello()` 方法。

### 步骤 1：创建过程宏 crate

```bash
cargo new hello_macro --lib
cargo new hello_macro_derive --lib
```

项目结构：
```
my_project/
├── hello_macro/           # 定义 trait
│   ├── Cargo.toml
│   └── src/lib.rs
├── hello_macro_derive/    # 实现过程宏
│   ├── Cargo.toml
│   └── src/lib.rs
└── my_app/                # 使用它们
    ├── Cargo.toml
    └── src/main.rs
```

### 步骤 2：定义 trait

```rust
// hello_macro/src/lib.rs
pub trait HelloMacro {
    fn hello();
}
```

### 步骤 3：实现过程宏

```toml
# hello_macro_derive/Cargo.toml
[lib]
proc-macro = true

[dependencies]
syn = "2"
quote = "1"
```

```rust
// hello_macro_derive/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // 解析输入的 Rust 代码
    let ast = parse_macro_input!(input as DeriveInput);
    
    // 获取结构体名称
    let name = &ast.ident;
    let name_str = name.to_string();
    
    // 生成实现代码
    let expanded = quote! {
        impl HelloMacro for #name {
            fn hello() {
                println!("Hello from {}!", #name_str);
            }
        }
    };
    
    TokenStream::from(expanded)
}
```

**解读：**
- `#[proc_macro_derive(HelloMacro)]` — 声明这是 HelloMacro 的派生宏
- `parse_macro_input!` — 把 token 流解析成 AST
- `quote!` — 用类似模板的语法生成代码
- `#name` — 在 quote! 里插入变量

### 步骤 4：使用它！

```rust
// my_app/src/main.rs
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

#[derive(HelloMacro)]
struct Waffles;

fn main() {
    Pancakes::hello();  // Hello from Pancakes!
    Waffles::hello();   // Hello from Waffles!
}
```

**这就是派生宏的魔力：** 编译器看到 `#[derive(HelloMacro)]`，就调用你的过程宏，自动生成 `impl HelloMacro for Pancakes { ... }`！

---

## 属性宏 (Attribute Macro)

比派生宏更灵活，可以修改任何项 (item)。

**示例：给函数添加日志**

```rust
// 使用
#[log_call]
fn add(a: i32, b: i32) -> i32 {
    a + b
}

// 展开后
fn add(a: i32, b: i32) -> i32 {
    println!("Calling add");
    let result = { a + b };
    println!("add returned {:?}", result);
    result
}
```

**实现：**

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemFn};

#[proc_macro_attribute]
pub fn log_call(
    _attr: TokenStream,  // 属性参数，如 #[log_call(level = "debug")]
    item: TokenStream,   // 被修饰的项
) -> TokenStream {
    let input = parse_macro_input!(item as ItemFn);
    
    let fn_name = &input.sig.ident;
    let fn_name_str = fn_name.to_string();
    let fn_block = &input.block;
    let fn_sig = &input.sig;
    let fn_vis = &input.vis;
    
    let expanded = quote! {
        #fn_vis #fn_sig {
            println!("Calling {}", #fn_name_str);
            let result = #fn_block;
            println!("{} returned {:?}", #fn_name_str, result);
            result
        }
    };
    
    TokenStream::from(expanded)
}
```

---

## 函数式过程宏 (Function-like Macro)

调用语法像函数，但在编译期执行。

**示例：SQL 校验宏**

```rust
// 编译期检查 SQL 语法！
let query = sql!(SELECT * FROM users WHERE id = ?);
```

**实现：**

```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
    let sql_str = input.to_string();
    
    // 这里可以做 SQL 语法校验
    // 如果语法错误，用 compile_error! 报错
    
    let expanded = quote! {
        #sql_str
    };
    
    TokenStream::from(expanded)
}
```

---

## syn 深入：解析结构体字段

派生宏最常见的需求是遍历结构体字段。

```rust
use syn::{parse_macro_input, DeriveInput, Data, Fields};

#[proc_macro_derive(Builder)]
pub fn builder_derive(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);
    let name = &ast.ident;
    
    // 获取字段
    let fields = match &ast.data {
        Data::Struct(data) => {
            match &data.fields {
                Fields::Named(fields) => &fields.named,
                _ => panic!("Builder only works on structs with named fields"),
            }
        }
        _ => panic!("Builder only works on structs"),
    };
    
    // 遍历每个字段
    for field in fields {
        let field_name = &field.ident;
        let field_type = &field.ty;
        // 生成对应代码...
    }
    
    // ...
}
```

---

## quote! 高级用法

### 重复生成

```rust
let field_names = vec!["a", "b", "c"];

let expanded = quote! {
    // #( ... )* 重复模式
    #(
        println!("Field: {}", #field_names);
    )*
};
// 生成：
// println!("Field: {}", "a");
// println!("Field: {}", "b");
// println!("Field: {}", "c");
```

### 条件生成

```rust
let is_debug = true;

let debug_code = if is_debug {
    quote! { println!("Debug mode"); }
} else {
    quote! {}
};

let expanded = quote! {
    fn init() {
        #debug_code
        // ...
    }
};
```

---

## 实战：实现 #[derive(Getters)]

自动为每个字段生成 getter 方法：

```rust
#[derive(Getters)]
struct User {
    name: String,
    age: u32,
}

// 自动生成
impl User {
    fn name(&self) -> &String { &self.name }
    fn age(&self) -> &u32 { &self.age }
}
```

**完整实现：**

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput, Data, Fields};

#[proc_macro_derive(Getters)]
pub fn getters_derive(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);
    let name = &ast.ident;
    
    let fields = match &ast.data {
        Data::Struct(data) => match &data.fields {
            Fields::Named(f) => &f.named,
            _ => panic!("Getters requires named fields"),
        },
        _ => panic!("Getters only works on structs"),
    };
    
    // 为每个字段生成 getter
    let getters = fields.iter().map(|f| {
        let field_name = f.ident.as_ref().unwrap();
        let field_type = &f.ty;
        
        quote! {
            pub fn #field_name(&self) -> &#field_type {
                &self.#field_name
            }
        }
    });
    
    let expanded = quote! {
        impl #name {
            #(#getters)*
        }
    };
    
    TokenStream::from(expanded)
}
```

---

## 调试过程宏

### 1. cargo expand

查看宏展开结果：

```bash
cargo install cargo-expand
cargo expand
```

### 2. eprintln! 调试

过程宏里的打印会在编译时输出：

```rust
#[proc_macro_derive(MyMacro)]
pub fn my_macro(input: TokenStream) -> TokenStream {
    eprintln!("INPUT: {:#?}", input);  // 编译时打印
    // ...
}
```

### 3. syn 的 Debug

```rust
let ast = parse_macro_input!(input as DeriveInput);
eprintln!("AST: {:#?}", ast);  // 打印完整语法树
```

---

## 常见过程宏库

| 库 | 用途 |
|----|------|
| `serde` | #[derive(Serialize, Deserialize)] |
| `thiserror` | #[derive(Error)] 自定义错误 |
| `tokio::main` | #[tokio::main] 异步入口 |
| `axum` | #[debug_handler] 路由调试 |
| `sqlx` | sqlx::query!() 编译时 SQL 检查 |
| `clap` | #[derive(Parser)] CLI 参数解析 |

---

## 声明式 vs 过程宏

| 特性 | macro_rules! | proc-macro |
|------|--------------|------------|
| 复杂度 | 简单 | 复杂 |
| 功能 | 模式替换 | AST 操作 |
| 依赖 | 无 | syn/quote |
| crate | 同一 crate | 单独 crate |
| 错误信息 | 较差 | 可自定义 |

**选择建议：**
- 简单模式替换 → `macro_rules!`
- 需要解析结构体/实现 trait → 派生宏
- 修饰函数/添加逻辑 → 属性宏

---

## 课后作业

实现 `#[derive(Debug2)]`，生成比标准 Debug 更可读的输出：

```rust
#[derive(Debug2)]
struct Point { x: i32, y: i32 }

// 生成的 debug2() 输出：
// Point {
//   x: 10
//   y: 20
// }
```

---

## 下节预告

下节课进入 **Unsafe Rust**：
- raw pointer 裸指针
- unsafe 块
- 什么时候需要 unsafe

---

*笔记整理：性奴001*
