# 第 100 课：std::mem — 内存操作核心工具

**里程碑达成！** 今天讲 Rust 最底层的内存操作模块。

---

## 📦 std::mem 是什么

`std::mem` 提供了直接操作内存布局和值的工具函数，是理解 Rust 内存模型的关键。

```rust
use std::mem;

// 常用函数一览
mem::size_of::<T>()      // 类型大小（字节）
mem::align_of::<T>()     // 类型对齐要求
mem::swap(&mut a, &mut b) // 交换两个值
mem::replace(&mut a, b)   // 替换并返回旧值
mem::take(&mut a)        // 取走值，留下 Default
mem::drop(x)             // 手动 drop（其实就是啥也不做）
mem::forget(x)           // 忘记值，不运行析构函数
mem::transmute(x)        // 危险：重新解释内存
```

---

## 📏 size_of 和 align_of — 了解内存布局

```rust
use std::mem;

// 基本类型大小
println!("i8:  {} bytes", mem::size_of::<i8>());   // 1
println!("i32: {} bytes", mem::size_of::<i32>());  // 4
println!("i64: {} bytes", mem::size_of::<i64>());  // 8
println!("f64: {} bytes", mem::size_of::<f64>());  // 8

// 指针和引用
println!("&i32:    {} bytes", mem::size_of::<&i32>());     // 8 (64位系统)
println!("Box<i32>: {} bytes", mem::size_of::<Box<i32>>()); // 8

// Option 的零成本抽象！
println!("Box<i32>:         {} bytes", mem::size_of::<Box<i32>>());         // 8
println!("Option<Box<i32>>: {} bytes", mem::size_of::<Option<Box<i32>>>()); // 8 也是！

// 因为 Box 不可能是 null，所以 None 可以用 null 表示
// 这叫「空指针优化」(Null Pointer Optimization)
```

### 结构体内存布局

```rust
struct A {
    a: u8,   // 1 byte
    b: u32,  // 4 bytes
    c: u8,   // 1 byte
}

struct B {
    a: u8,   // 1 byte
    c: u8,   // 1 byte  
    b: u32,  // 4 bytes
}

println!("A: {} bytes", mem::size_of::<A>()); // 12！不是 6
println!("B: {} bytes", mem::size_of::<B>()); // 8

// 为什么？内存对齐！
// A 的布局: [a:1][padding:3][b:4][c:1][padding:3] = 12
// B 的布局: [a:1][c:1][padding:2][b:4] = 8

// Rust 编译器会自动重排字段优化大小（除非你用 #[repr(C)]）
```

### 对齐要求

```rust
println!("align of u8:  {}", mem::align_of::<u8>());   // 1
println!("align of u32: {}", mem::align_of::<u32>());  // 4
println!("align of u64: {}", mem::align_of::<u64>());  // 8

// 结构体的对齐 = 最大字段的对齐
struct Mixed {
    a: u8,
    b: u64,
}
println!("align of Mixed: {}", mem::align_of::<Mixed>()); // 8
```

---

## 🔄 swap — 交换两个值

```rust
use std::mem;

let mut a = 1;
let mut b = 2;

mem::swap(&mut a, &mut b);

println!("a = {}, b = {}", a, b); // a = 2, b = 1

// 为什么需要 mem::swap？
// 因为你不能这样写：
// let temp = a;  // ❌ a 被 move 了
// a = b;         // ❌ b 被 move 了
// b = temp;

// 对于 Copy 类型可以用 std::mem::swap 或直接赋值
// 对于非 Copy 类型，mem::swap 是最清晰的方式
```

### 实际应用：在链表中交换节点

```rust
struct Node {
    value: i32,
    next: Option<Box<Node>>,
}

fn swap_values(a: &mut Node, b: &mut Node) {
    mem::swap(&mut a.value, &mut b.value);
}
```

---

## 🔁 replace — 替换并返回旧值

```rust
use std::mem;

let mut v = vec![1, 2, 3];

// 取走 v，放入空 Vec，返回原来的值
let old = mem::replace(&mut v, Vec::new());

println!("old: {:?}", old); // [1, 2, 3]
println!("v: {:?}", v);     // []

// 常见用法：从 &mut self 中取走某个字段
struct Parser {
    buffer: String,
}

impl Parser {
    fn take_buffer(&mut self) -> String {
        mem::replace(&mut self.buffer, String::new())
    }
}
```

---

## 📤 take — replace 的简化版

```rust
use std::mem;

let mut s = String::from("hello");

// take 等价于 replace(&mut s, Default::default())
let taken = mem::take(&mut s);

println!("taken: {}", taken); // hello
println!("s: '{}'", s);       // '' (空字符串)

// 要求类型实现 Default trait
// Vec::default() = vec![]
// String::default() = ""
// Option::default() = None
```

### take 的经典用法

```rust
struct State {
    data: Option<Vec<i32>>,
}

impl State {
    fn process(&mut self) {
        // 取走 data 进行处理，不需要 clone
        if let Some(data) = mem::take(&mut self.data) {
            for x in data {
                println!("{}", x);
            }
        }
        // self.data 现在是 None
    }
}
```

---

## 🗑️ drop — 提前释放资源

```rust
use std::mem;

{
    let v = vec![1, 2, 3];
    
    // 想在作用域结束前就释放？
    mem::drop(v);  // 等价于 drop(v)
    
    // v 已经被 drop，不能再用了
    // println!("{:?}", v); // ❌ 编译错误
}

// 其实 mem::drop 的实现超级简单：
// pub fn drop<T>(_x: T) { }
// 它什么都不做！只是获取所有权，然后函数结束时自然 drop
```

### drop 的实际用途

```rust
use std::sync::Mutex;

let mutex = Mutex::new(42);

{
    let guard = mutex.lock().unwrap();
    println!("locked: {}", *guard);
    
    // 想提前释放锁？
    drop(guard);  // 锁在这里释放
    
    // 可以继续做不需要锁的事情
    println!("lock released, doing other work...");
}
```

---

## ⚠️ forget — 不运行析构函数（危险！）

```rust
use std::mem;

let v = vec![1, 2, 3];

// forget 会「忘记」这个值，不调用 Drop
mem::forget(v);

// 内存泄漏了！Vec 的堆内存没有被释放
// 这是 safe Rust 中少数几个能造成内存泄漏的方式
```

### forget 的合法用途

```rust
// 1. FFI：把所有权转移给 C 代码
extern "C" fn give_to_c(ptr: *mut i32, len: usize);

let mut v = vec![1, 2, 3];
let ptr = v.as_mut_ptr();
let len = v.len();

// 告诉 C 代码负责释放
give_to_c(ptr, len);

// Rust 不要再管这块内存了
mem::forget(v);

// 2. ManuallyDrop 是更好的选择
use std::mem::ManuallyDrop;

let v = ManuallyDrop::new(vec![1, 2, 3]);
// v 不会自动 drop，但你可以手动 drop
// ManuallyDrop::drop(&mut v);
```

---

## 🔀 transmute — 重新解释内存（超级危险！）

```rust
use std::mem;

// transmute 把一种类型的位模式强制解释为另一种类型
// ⚠️ 这是 unsafe 的，用错会导致 UB

unsafe {
    let x: u32 = 0x41424344;
    let bytes: [u8; 4] = mem::transmute(x);
    println!("{:?}", bytes); // [68, 67, 66, 65] 小端序
    
    // 更安全的替代方案
    let bytes = x.to_ne_bytes(); // 原生字节序
    let bytes = x.to_le_bytes(); // 小端
    let bytes = x.to_be_bytes(); // 大端
}
```

### transmute 的危险

```rust
// ❌ 绝对不要这样做！
unsafe {
    let s = String::from("hello");
    let v: Vec<u8> = mem::transmute(s); // UB！布局可能不同
}

// ✅ 正确方式
let s = String::from("hello");
let v: Vec<u8> = s.into_bytes();
```

---

## 🧠 MaybeUninit — 安全处理未初始化内存

```rust
use std::mem::MaybeUninit;

// 创建未初始化的值
let mut x: MaybeUninit<i32> = MaybeUninit::uninit();

// 写入值
x.write(42);

// 安全地取出（你必须确保已初始化）
let x: i32 = unsafe { x.assume_init() };
println!("{}", x); // 42

// 数组的延迟初始化
let mut arr: [MaybeUninit<String>; 3] = unsafe {
    MaybeUninit::uninit().assume_init()
};

arr[0].write(String::from("a"));
arr[1].write(String::from("b"));
arr[2].write(String::from("c"));

// 转换为初始化的数组
let arr: [String; 3] = unsafe {
    // transmute 在这里是安全的，因为我们确保都初始化了
    std::mem::transmute(arr)
};
```

---

## 💡 实用技巧总结

```rust
use std::mem;

// 1. 获取值的大小（运行时）
let v = vec![1, 2, 3];
println!("size: {}", mem::size_of_val(&v)); // Vec 结构本身的大小

// 2. 安全地从 Option 取值
let mut opt = Some(String::from("hello"));
let val = opt.take(); // 等价于 mem::replace(&mut opt, None)

// 3. 清空并获取 Vec 内容
let mut v = vec![1, 2, 3];
let old = mem::take(&mut v); // v 变成 []

// 4. 判断类型是否为零大小类型 (ZST)
println!("() is ZST: {}", mem::size_of::<()>() == 0); // true
println!("[u8; 0] is ZST: {}", mem::size_of::<[u8; 0]>() == 0); // true
```

---

## 🎓 课后思考

1. 为什么 `Option<Box<T>>` 和 `Box<T>` 大小相同？
2. `mem::forget` 是 safe 函数，为什么造成内存泄漏却是安全的？
3. 什么时候用 `mem::replace` vs `mem::take`？

---

*🎊 第 100 课完 — 我们已经走过了 100 节课的 Rust 之旅！*
