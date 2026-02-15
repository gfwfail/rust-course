# 第11课：Trait 对象与动态分发（dyn Trait）

上节课学了 Trait 作为参数约束（静态分发），今天学**运行时**决定调用哪个实现。

## 什么场景需要 Trait 对象？

假设我们有一个绘图程序，需要存储不同形状：

```rust
trait Draw {
    fn draw(&self);
}

struct Circle { radius: f64 }
struct Rectangle { width: f64, height: f64 }

impl Draw for Circle {
    fn draw(&self) {
        println!("画一个半径 {} 的圆", self.radius);
    }
}

impl Draw for Rectangle {
    fn draw(&self) {
        println!("画一个 {}x{} 的矩形", self.width, self.height);
    }
}
```

如何把 Circle 和 Rectangle 放进同一个 Vec？

```rust
// ❌ 编译错误！Vec 只能存一种类型
let shapes: Vec<???> = vec![Circle {...}, Rectangle {...}];
```

## Trait 对象：`dyn Trait`

**Trait 对象** 允许我们存储「任何实现了这个 Trait 的类型」：

```rust
// ✅ 用 Box<dyn Draw> 存储不同类型
let shapes: Vec<Box<dyn Draw>> = vec![
    Box::new(Circle { radius: 5.0 }),
    Box::new(Rectangle { width: 10.0, height: 3.0 }),
];

// 遍历并调用——运行时才决定调哪个 draw()
for shape in &shapes {
    shape.draw();
}
```

**关键点：**
- `dyn Draw` 表示「任何实现了 Draw 的类型」
- 必须放在指针后面：`Box<dyn Draw>`、`&dyn Draw`、`Rc<dyn Draw>` 等
- 不能直接用 `dyn Draw`（编译器不知道大小）

## 为什么必须用 Box？

Trait 对象的大小在编译时未知——Circle 和 Rectangle 大小不同！

```rust
// ❌ 编译错误：dyn Draw 大小未知
let x: dyn Draw = Circle { radius: 5.0 };

// ✅ Box 放堆上，指针大小固定
let x: Box<dyn Draw> = Box::new(Circle { radius: 5.0 });

// ✅ 引用也行（指针大小固定）
fn render(shape: &dyn Draw) {
    shape.draw();
}
```

## 静态分发 vs 动态分发

| 对比 | 静态分发 `impl Trait` | 动态分发 `dyn Trait` |
|------|----------------------|---------------------|
| 决定时机 | 编译时 | 运行时 |
| 性能 | 更快（可内联） | 有虚函数表开销 |
| 灵活性 | 单一类型 | 多种类型混合存储 |
| 代码体积 | 每种类型生成一份 | 共享一份代码 |

```rust
// 静态分发：编译时就知道是什么类型
fn draw_static(shape: &impl Draw) {
    shape.draw();
}

// 动态分发：运行时通过虚函数表查找
fn draw_dynamic(shape: &dyn Draw) {
    shape.draw();
}
```

## 对象安全（Object Safety）

不是所有 Trait 都能变成 Trait 对象！必须满足 **对象安全** 规则：

```rust
// ✅ 对象安全：方法只用 &self
trait Draw {
    fn draw(&self);
}

// ❌ 不对象安全：方法返回 Self
trait Clone {
    fn clone(&self) -> Self;  // Self 大小未知！
}

// ❌ 不对象安全：有泛型方法
trait BadTrait {
    fn foo<T>(&self, x: T);  // 无限多个可能的 T！
}
```

**简单规则：**
- 方法不能返回 `Self`
- 方法不能有泛型参数

## 实际例子：回调函数存储

```rust
type Callback = Box<dyn Fn(i32) -> i32>;

fn main() {
    let callbacks: Vec<Callback> = vec![
        Box::new(|x| x + 1),
        Box::new(|x| x * 2),
        Box::new(|x| x * x),
    ];
    
    let value = 5;
    for (i, cb) in callbacks.iter().enumerate() {
        println!("回调{}: {} -> {}", i, value, cb(value));
    }
}
```

---

**下节课**：Vec 动态数组
