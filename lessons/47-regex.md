# 第 47 课：正则表达式 (regex)

> 日期：2026-02-18  
> 主题：使用 regex crate 进行模式匹配、文本搜索和替换

---

## 📌 本课目标

学会用 Rust 的 `regex` crate 进行模式匹配、文本搜索和替换。

---

## 为什么要学 regex crate？

PHP/JS 正则是内置的：
```php
preg_match('/\d+/', $text, $matches);
```

Rust 标准库**没有**内置正则！需要用 `regex` crate：
- 零运行时依赖
- 编译时优化正则
- 保证线性时间复杂度（不会有 ReDoS 攻击）

---

## 添加依赖

```toml
[dependencies]
regex = "1"
```

---

## 基础用法：检查是否匹配

```rust
use regex::Regex;

fn main() {
    let re = Regex::new(r"\d{4}-\d{2}-\d{2}").unwrap();
    
    // is_match: 检查是否匹配
    println!("{}", re.is_match("2026-02-18")); // true
    println!("{}", re.is_match("not a date")); // false
}
```

⚠️ **注意**：用 `r"..."` raw string，避免转义地狱！

---

## 查找匹配：find 和 find_iter

```rust
use regex::Regex;

fn main() {
    let re = Regex::new(r"\d+").unwrap();
    let text = "价格：100 元，数量：5 个";
    
    // find: 找第一个匹配
    if let Some(m) = re.find(text) {
        println!("找到: {} (位置 {}..{})", 
            m.as_str(), m.start(), m.end());
        // 找到: 100 (位置 9..12)
    }
    
    // find_iter: 找所有匹配
    for m in re.find_iter(text) {
        println!("{}", m.as_str());
    }
    // 100
    // 5
}
```

---

## 捕获组：captures

```rust
use regex::Regex;

fn main() {
    let re = Regex::new(r"(\d{4})-(\d{2})-(\d{2})").unwrap();
    let text = "日期是 2026-02-18 今天";
    
    if let Some(caps) = re.captures(text) {
        println!("完整匹配: {}", &caps[0]); // 2026-02-18
        println!("年: {}", &caps[1]);        // 2026
        println!("月: {}", &caps[2]);        // 02
        println!("日: {}", &caps[3]);        // 18
    }
}
```

---

## 命名捕获组

```rust
use regex::Regex;

fn main() {
    let re = Regex::new(
        r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})"
    ).unwrap();
    
    if let Some(caps) = re.captures("2026-02-18") {
        println!("年: {}", &caps["year"]);
        println!("月: {}", &caps["month"]);
        println!("日: {}", &caps["day"]);
    }
}
```

命名捕获组用 `(?P<name>...)` 语法，取值用 `&caps["name"]`。

---

## 文本替换：replace

```rust
use regex::Regex;

fn main() {
    let re = Regex::new(r"\d+").unwrap();
    
    // replace: 只替换第一个
    let result = re.replace("a1b2c3", "X");
    println!("{}", result); // aXb2c3
    
    // replace_all: 替换所有
    let result = re.replace_all("a1b2c3", "X");
    println!("{}", result); // aXbXcX
}
```

---

## 用捕获组替换

```rust
use regex::Regex;

fn main() {
    let re = Regex::new(r"(\w+)@(\w+)\.com").unwrap();
    
    // $1, $2 引用捕获组
    let result = re.replace_all(
        "联系 test@example.com 或 hello@world.com",
        "[$1 at $2]"
    );
    println!("{}", result);
    // 联系 [test at example] 或 [hello at world]
}
```

---

## 性能优化：编译一次，多次使用

❌ **反面教材**：每次都编译正则
```rust
fn is_email(s: &str) -> bool {
    // 每次调用都编译，超慢！
    Regex::new(r"^\S+@\S+$").unwrap().is_match(s)
}
```

✅ **正确做法**：用 `LazyLock`（Rust 1.80+）

```rust
use regex::Regex;
use std::sync::LazyLock;

static EMAIL_RE: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r"^\S+@\S+$").unwrap()
});

fn is_email(s: &str) -> bool {
    EMAIL_RE.is_match(s)
}

fn main() {
    println!("{}", is_email("test@example.com")); // true
    println!("{}", is_email("not an email"));     // false
}
```

`LazyLock` 是 Rust 1.80+ 标准库新增的，替代 `lazy_static!` 宏。

---

## 实战：解析日志

```rust
use regex::Regex;
use std::sync::LazyLock;

static LOG_RE: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(
        r#"(?P<ip>\d+\.\d+\.\d+\.\d+) .* \[(?P<time>[^\]]+)\] "(?P<method>\w+) (?P<path>[^ ]+)"#
    ).unwrap()
});

fn parse_log(line: &str) -> Option<(String, String, String, String)> {
    LOG_RE.captures(line).map(|caps| {
        (
            caps["ip"].to_string(),
            caps["time"].to_string(),
            caps["method"].to_string(),
            caps["path"].to_string(),
        )
    })
}

fn main() {
    let log = r#"192.168.1.1 - - [18/Feb/2026:15:00:00 +1100] "GET /api/users HTTP/1.1" 200"#;
    
    if let Some((ip, time, method, path)) = parse_log(log) {
        println!("IP: {}", ip);
        println!("Time: {}", time);
        println!("Method: {}", method);
        println!("Path: {}", path);
    }
}
```

---

## 对比其他语言

| 操作 | PHP | Rust regex |
|------|-----|------------|
| 检查匹配 | `preg_match()` | `re.is_match()` |
| 找第一个 | `preg_match()` | `re.find()` |
| 找所有 | `preg_match_all()` | `re.find_iter()` |
| 捕获组 | `preg_match($pat, $s, $m)` | `re.captures()` |
| 替换 | `preg_replace()` | `re.replace()` |

---

## 注意事项

1. **编译失败不要 unwrap**（用户输入时）
```rust
match Regex::new(user_input) {
    Ok(re) => { /* 使用 */ }
    Err(e) => println!("无效正则: {}", e),
}
```

2. **复杂正则用 verbose 模式**
```rust
let re = Regex::new(r"(?x)
    (?P<year>\d{4})   # 年份
    -
    (?P<month>\d{2})  # 月份
    -
    (?P<day>\d{2})    # 日期
").unwrap();
```

3. **不要过度使用正则**
简单场景用 `str::contains`、`str::starts_with` 更快！

---

## 课后小练习

写一个函数，从文本中提取所有手机号（假设格式为 1 开头的 11 位数字）：

```rust
fn extract_phones(text: &str) -> Vec<&str> {
    // 你的代码
}

// 测试
// extract_phones("联系 13812345678 或 15987654321")
// => ["13812345678", "15987654321"]
```

---

*下节课预告：rand 随机数生成*
