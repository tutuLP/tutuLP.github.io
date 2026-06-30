---
title: rust基础
date: 2025-08-04
categories:
  - rust
---

# 环境

下载rustup
https://www.rust-lang.org/zh-CN/learn/get-started

- windows网址中下载
- Linux：  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
- * 更新bash环境变量：source $HOME/.cargo/env

查看安装版本

- rustup show
- cargo --version   cargo是rust的包管理工具，类似python的pip
- rustc --version

将vs中c++的环境添加到环境变量

C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.42.34433\bin\Hostx64\x64





'static 生命周期

1. 字符串字面值就具有 'static 生命周期
   函数参数类型：&'static str ，传入参数类型&str
2. 特征对象的生命周期



```
assert_eq!(a, b);
```

基本类型已经实现了`Debug` trait和Display trait所以可以直接使用{:?} {}

## 编辑器

* clion

rust插件：https://plugins.jetbrains.com/plugin/8182--deprecated-rust

* vscode

插件：rust-analyzer

# start

main.rs

```rust
fn main() {
    println!("Hello, world!");
}
```

## 格式化输出

std::fmt中的宏

format!：将格式化文本写到字符串。
print!：将文本输出到控制台（io::stdout）。
println!: 与 print! 类似，但输出结果追加一个换行符。
eprint!：与 print! 类似，但将文本输出到标准错误（io::stderr）。
eprintln!：与 eprint! 类似，但输出结果追加一个换行符。

### format

println!("{} days", 31i64);  //{}直接替换为后面的内容，整型或者""字符串，31默认i32，可以手动指定为31i64

println!("{0}, this is {1}. {1}, this is {0}", "Alice", "Bob");   //使用{0}{1},替换多个变量

println!("{subject}",subject="the lazy dog");   //使用命名参数替换

println!("{number:>width\$}", number=1, width=6);   //指定宽度对齐 左对齐 :< ，前面加0：:>0width$

{:.2} //小数

> #[allow(dead_code)]  //忽略死代码片段：未使用的代码段
> struct Structure(i32);

- `{}` 对应 `fmt::Display` `trait`，它定义了类型的常规显示格式。
- `{:?}` 对应 `fmt::Debug` `trait`，它定义了类型的调试信息显示格式。
- `{:b}` 对应 `fmt::Binary` `trait`，它定义了类型的二进制表示格式。

### 调式debug

```~rust
struct UnPrintable(i32); //不能直接使用`fmt::Display` 或 `fmt::Debug` 来进行打印

#[derive(Debug)] 
struct DebugPrintable(i32);//`derive` 属性会自动创建所需的实现，使这个 `struct` 能使用 `fmt::Debug` 打印
```~

用{:?}打印
使用和{}一样：{1:?} {0:?} {actor:?}

美化打印 {:#?}

```rust
#[derive(Debug)]
struct Person<'a> {
    name: &'a str,
    age: u8
}
fn main() {
    let name = "Peter";
    let age = 27;
    let peter = Person { name, age };
    // 美化打印
    println!("{:#?}", peter);
}
//Person {
//    name: "Peter",
//    age: 27,
//}
```

### 显示Display

display不像debug声明#[derive(Debug)]后就会自动提供实现，display必须手动为类型实现 fmt::Display trait，display更简洁，导致Vec<T> 或其他任意泛型容器用用 fmt::Debug 而不用Display，因为输出样式不同

```rust
use std::fmt;  //导入模块使`fmt::Display` 可用
struct Structure(i32);  
impl fmt::Display for Structure {  //impl：为 Structure 结构体实现 fmt::Display trait
    // 这个 trait 要求 `fmt` 使用与下面的函数完全一致的函数签名
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        // 仅将 self 的第一个元素写入到给定的输出流 `f`。返回 `fmt:Result`，此结果表明操作成功或失败。注意 `write!` 的用法和 `println!` 很相似。
        write!(f, "{}", self.0)
    }
}
```

对比Display和debug

```rust
use std::fmt; 
#[derive(Debug)]  //Debug
struct MinMax(i64, i64);
impl fmt::Display for MinMax {  //Display
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.0, self.1)
    }
}

#[derive(Debug)]
struct Point2D {
    x: f64,
    y: f64,
}
impl fmt::Binary for Point2D{   //Binary  {:b}
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let x_bits = self.x.to_bits();  //Binary只对整型，转换为整型
        let y_bits = self.y.to_bits();
        write!(f, "{:b}{:b}", x_bits, y_bits)
    }
}

impl fmt::Display for Point2D {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "x: {}, y: {}", self.x, self.y)
    }
}

fn main() {
    let minmax = MinMax(0, 14);

    println!("Compare structures:");
    println!("Display: {}", minmax);
    println!("Debug: {:?}", minmax);

    let big_range =   MinMax(-300, 300);
    let small_range = MinMax(-3, 3);

    println!("The big range is {big} and the small is {small}",small = small_range,big = big_range);

    let point = Point2D { x: 3.3, y: 7.2 };

    println!("Compare points:");
    println!("Display: {}", point);
    println!("Debug: {:?}", point);
    println!("What does Point2D look like in binary: {:b}?", point);
}
```

### 测试实例List

write!配合?操作符实现复杂的拼接，因为使用write都会生成一个Resualt，这里使用?就如果成功就继续，是被则从fmt返回错误，减少了if的判断

```rust
use std::fmt; // 导入 `fmt` 模块。

// 定义一个包含单个 `Vec` 的结构体 `List`。
struct List(Vec<i32>);

impl fmt::Display for List {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        // 使用元组的下标获取值，并创建一个 `vec` 的引用。
        let vec = &self.0;

        write!(f, "[")?;

        // 使用 `v` 对 `vec` 进行迭代，并用 `count` 记录迭代次数。
        for (count, v) in vec.iter().enumerate() {
            // 对每个元素（第一个元素除外）加上逗号。
            // 使用 `?` 或 `try!` 来返回错误。
            if count != 0 { write!(f, ", ")?; }
            write!(f, "{}", v)?;
        }

        // 加上配对中括号，并返回一个 fmt::Result 值。
        write!(f, "]")
    }
}

fn main() {
    let v = List(vec![1, 2, 3]);
    println!("{}", v);
}
```

### 格式化

```
- `format!("{}", foo)` -> `"3735928559"`
- `format!("0x{:X}", foo)` -> [`"0xDEADBEEF"`](https://en.wikipedia.org/wiki/Deadbeef#Magic_debug_values)
- `format!("0o{:o}", foo)` -> `"0o33653337357"`
```

```rust
use std::fmt::{self, Formatter, Display};

struct City {
    name: &'static str,
    // 纬度
    lat: f32,
    // 经度
    lon: f32,
}

impl Display for City {
    // `f` 是一个缓冲区（buffer），此方法必须将格式化后的字符串写入其中
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        let lat_c = if self.lat >= 0.0 { 'N' } else { 'S' };
        let lon_c = if self.lon >= 0.0 { 'E' } else { 'W' };

        // `write!` 和 `format!` 类似，但它会将格式化后的字符串写入
        // 一个缓冲区（即第一个参数f）中。
        write!(f, "{}: {:.3}°{} {:.3}°{}",
               self.name, self.lat.abs(), lat_c, self.lon.abs(), lon_c)
    }
}

#[allow(dead_code)]
#[derive(Debug)]
struct Color {
    red: u8,
    green: u8,
    blue: u8,
}

fn main() {
    for city in [
        City { name: "Dublin", lat: 53.347778, lon: -6.259722 },
        City { name: "Oslo", lat: 59.95, lon: 10.75 },
        City { name: "Vancouver", lat: 49.25, lon: -123.1 },
    ].iter() {
        println!("{}", *city);
    }
    for color in [
        Color { red: 128, green: 255, blue: 90 },
        Color { red: 0, green: 3, blue: 254 },
        Color { red: 0, green: 0, blue: 0 },
    ].iter() {
        println!("{:?}", *color);
        println!("Red: {}, Green: {}, Blue: {}", color.red, color.green, color.blue);
    }
}
```

# 原生类型

### 标量类型

有符号整数：i8 i16 i32 i64 i128 isize(指针宽度)
无符号整数：u8 u16 u32 u64 u128 usize
浮点数：f32 f64
char: 'a' 'α' '∞' 都是四字节
bool：ture/false
单元类型：()  唯一的值就是这个空元组

> 尽管单元类型的值是个元组，它却并不被认为是复合类型，因为并不包含多个值。

### 复合类型

数组：[1,2,3]
元组：(1，true)



整型默认为 `i32` 类型，浮点型默认为 `f64`类型。
根据上下文环境自动推断：未声明类型整数+i64，该整数会自动推断为i64，无法推断时按默认值处理

```rust
let logical: bool = true; //常规说明
let an_integer   = 5i32; //后缀说明
let default_float   = 3.0;//默认
let mut mutable = 12;//可变值，但是变量的类型不能改变
```

## 字面量和运算符

通过加前缀 `0x`、`0o`、`0b`，数字可以用十六进制、八进制或二进制记法表示。
可以在数值字面量中插入下划线，比如：`1_000` 等同于 `1000`，`0.000_001` 等同于 `0.000001`。

```rust
fn main() {
    // 整数相加
    println!("1 + 2 = {}", 1u32 + 2);

    // 整数相减
    println!("1 - 2 = {}", 1i32 - 2);
    // 试一试 ^ 尝试将 `1i32` 改为 `1u32`，体会为什么类型声明这么重要

    // 短路求值的布尔逻辑
    println!("true AND false is {}", true && false);
    println!("true OR false is {}", true || false);
    println!("NOT true is {}", !true);

    // 位运算
    println!("0011 AND 0101 is {:04b}", 0b0011u32 & 0b0101);
    println!("0011 OR 0101 is {:04b}", 0b0011u32 | 0b0101);
    println!("0011 XOR 0101 is {:04b}", 0b0011u32 ^ 0b0101);
    println!("1 << 5 is {}", 1u32 << 5);
    println!("0x80 >> 2 is 0x{:x}", 0x80u32 >> 2);

    // 使用下划线改善数字的可读性！
    println!("One million is written as {}", 1_000_000u32);
}
```

## 元组

元组是一个可以包含各种类型值的组合，可以拥有任意个值

```rust
fn reverse(pair: (i32, bool)) -> (bool, i32) { //参数：pair元组，返回值也是元组
    let (integer, boolean) = pair;//赋值
    (boolean, integer)//返回值
}

#[derive(Debug)]
struct Matrix(f32, f32, f32, f32);

fn main() {
    // 包含各种不同类型的元组
    let long_tuple = (1u8, 2u16, 3u32, 4u64,
                      -1i8, -2i16, -3i32, -4i64,
                      0.1f32, 0.2f64,
                      'a', true);

    // 通过元组的下标来访问具体的值
    println!("long tuple first value: {}", long_tuple.0);

    // 元组也可以充当元组的元素
    let tuple_of_tuples = ((1u8, 2u16, 2u32), (4u64, -1i8), -2i16);

    // 元组可以打印
    println!("tuple of tuples: {:?}", tuple_of_tuples);// 但很长的元组无法打印

    let pair = (1, true);
    println!("the reversed pair is {:?}", reverse(pair));

    // 创建单元素元组需要一个额外的逗号，这是为了和被括号包含的字面量作区分。
    println!("one element tuple: {:?}", (5u32,));
    println!("just an integer: {:?}", (5u32));

    // 元组可以被解构（deconstruct），从而将值绑定给变量
    let tuple = (1, "hello", 4.5, true);

    let (a, b, c, d) = tuple;
    println!("{:?}, {:?}, {:?}, {:?}", a, b, c, d);

    let matrix = Matrix(1.1, 1.2, 2.1, 2.2);
    println!("{:?}", matrix)//打印会打印出结构体的名字

}
```

## 数组/向量/切片

> 切片可以切数组、向量、String

### 初始化

```rust
//数组初始化 - 必须在编译时确定长度
let arr = [10, 20, 30, 40, 50];
let arr: [u8; 256] = [0; 256];

// 向量初始化
let mut vec: Vec<i32> = Vec::new();
let vec = vec![0; 100]; 
let mut rows = vec![String::new(); 3];

// 切片，表示截取arr下标从1到3
let slice = &arr[1..4];
```

### 向量操作

```rust
// 遍历
for (idx, word) in words.iter().enumerate() {}
```



## 字符串String/&str

* 遍历

```rust
for ch in s.chars() {
    println!("{}", ch);
}

for (i, ch) in s.chars().enumerate() {
    println!("{} {}", i, ch);
}

// 遍历ASCLL字符串，性能最高
for b in s.as_bytes() {
    println!("{}", *b as char);
}

// 需要访问
for i in 0..s.len(){

}
```



### 字符数组-Vec<char>

* 初始化

```rust
let chars = vec!['a', 'b', 'c'];

// String转化为Vec<char>
let chars: Vec<char> = s.chars().collect();
// 无法直接对String进行下标访问，需要先转化为定长字符数组

// String转化为ASCII字符数组
let bytes = s.as_bytes(); 

// vec转化为String
let str = result.into_iter().collect();
```



* 遍历

```rust
for ch in &chars {
    println!("{}", ch);
}

//需要索引
for (i, ch) in chars.iter().enumerate() {
    println!("{} {}", i, ch);
}

// 频繁下标访问
for i in 0..chars.len() {
    println!("{}", chars[i]);
}
```

### 字符串操作

* 去除空格，获取每个单词

```rust
for (idx, word) in s.split_whitespace().enumerate() {} //word的类型为&str

// 如果不需要索引
 for word in s.split_whitespace() {}

// 收集到向量
let words: Vec<&str> = s.split_whitespace().collect(); 
for (idx, word) in words.iter().enumerate() {} //word 的类型为&&str，需要*解引用访问
```



### 字符操作

* 转化为字面量（ASCLL）
  `b'A'`
  `'A' as u8`
* 字面量转化为字符
  `s[i] as char`

* 将字符串去除标点并转化为小写

对&str和String的操作相同，操作String会自动解引用

is_ascii_punctuation 判断是否属于 ASCII 标点
is_alphanumeric 判断是否为字母或数字

法一性能更好，虽然理论上法二只产生一个新的String，但是多出了很多迭代器并且频繁push，并且rust内部优化很好

```rust
// 法一
let str = input.to_lowercase().chars().filter(|c| c.is_alphanumeric()).collect();
// to_lowercase 产生新 String
// collect 产生新 String

// 法二
fn normalize(input: &str) -> String {
    let mut out = String::with_capacity(input.len()); //申请一段堆内存

    for c in input.chars().flat_map(|c| c.to_lowercase()) {
        if c.is_alphanumeric() {
            out.push(c);
        }
    }

    out
}
```

## 哈希表

### 初始化

`use std::collections::HashMap;`

```rust
// 初始化一个空的map
let mut map: HashMap<u8, &str> = HashMap::new();

// 直接初始化
let roman = HashMap::from([
    (b'I', 1),
    (b'V', 5),
    (b'X', 10),
    (b'L', 50),
    (b'C', 100),
    (b'D', 500),
    (b'M', 1000),
]);
// 访问
let p = b'I';
let x = roman[&p];

// use rustc_hash::FxHashMap;
let roman: FxHashMap<u8, i32> = FxHashMap::from_iter([
    (b'I', 1),
    (b'V', 5),
    (b'X', 10),
    (b'L', 50),
    (b'C', 100),
    (b'D', 500),
    (b'M', 1000),
]);

if let Some(&x) = roman.get(&b'X') {
    println!("{x}");
} else {
    eprintln!("Roman numeral 'X' not found");
}

let x = *roman.get(&b'X').ok_or_else(|| anyhow!("Roman numeral 'X' not found"))?;
```

### 操作

```rust
// 访问-注意访问不到可能unwrap
let x = roman[&p];

if let Some(&x) = roman.get(&b'X') {
    println!("{x}");
} else {
    eprintln!("Roman numeral 'X' not found");
}

// 查看某个key是否存在
if map.contains_key(&pat[idx]) {}

//判断某个value是否存在
if !map.values().any(|&v| v == *word) { 
    map.insert(pat[idx],word);
}
```



# clippy

> 代码规范检查

```shell
# 验证是否安装，和rustup一起提供的
cargo clippy -V

# 运行检查
cargo clippy --all-targets --all-features -- \
  --deny warnings \
  --deny clippy::unwrap_used \
  --deny clippy::expect_used \
  --deny clippy::let_underscore_must_use
  
# 需要跳过检查的部分
#[allow(clippy::expect_used)]
```



# anyhow

```rust
use anyhow::{Result, anyhow};

fn foo() -> Result<()> {
    Ok(())
}
```

## 使用

### contex/with_context

错误向上传播并且用contex说明该错误，会保留原有错误链

```rust
let content = std::fs::read_to_string("config.toml")
    .context("failed to read config.toml")?;

// 使用变量
let content = std::fs::read_to_string(&path)
    .with_context(|| format!("failed to read file: {}", path))?;
```

效果：


```
Error: init global resources failed # 最外层context

Caused by:
    0: failed to create mysql pool # 内层context
    1: pool timed out while waiting for an open connection # 原始错误
```

### map_err

丢弃原有错误链

```rust
// 丢弃原错误
let n: i32 = s.parse()
    .map_err(|_| anyhow!("invalid number: {}", s))?;

let n: i32 = s.parse()
    .map_err(|e| anyhow!("parse int failed: {}, err={}", s, e))?;

```

### ok_or / ok_or_else

处理Option

```rust
// None
let user = users.get(&uid)
    .ok_or_else(|| anyhow!("user not found: {}", uid))?;
```

### return Err/bail!

手动返回错误

```rust
if config.redis_url.is_empty() {
    return Err(anyhow!("redis_url is empty"));
}
// 等价
if config.redis_url.is_empty() {
    bail!("redis_url is empty");
}
```

## 调用处理

1. 使用?传递
2. 返回值为Result<()>,直接使用`if let Err(e) =`
3. 使用默认值

* unwrap_or() 慎用会跳过错误处理

* unwrap_or_else 可以使用,但是也要错误处理

```
unwrap_or_else(|e| {
    log::warn!("A failed: {}", e);
    0
});
```

* match 匹配

```
let v = match A() {
    Ok(v) => v,
    Err(e) => {
        log::error!("A error: {}", e);
        return Err(e);
    }
};
```





# warn

* the following packages contain code that will be rejected by a future version of Rust: redis v0.25.4

升级依赖redis的版本

* warning: `/Users/xzmini11/.cargo/config` is deprecated in favor of `config.toml`

mv ~/.cargo/config ~/.cargo/config.toml

# 多项目-creat管理

cargo new my_lib --lib

方法一

```
workspace-root/
├── project/        
│   ├── Cargo.toml
│   └── src/
│
├── embedding-lib/    ← crate
│   ├── Cargo.toml
│   └── src/

[dependencies]
embedding-lib = { path = "../embedding-lib" } #在 project 里直接引用
```

方法二-workspace

```
project/
├── Cargo.toml          ← workspace
├── embedding-lib/    
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       └── embedding.rs
│
├── app/                ← 原来的项目
│   ├── Cargo.toml
│   └── src/
│       ├── api/
│       ├── dao/
│       ├── domain/
│       ├── dto/
│       ├── infra/
│       ├── service/
│       └── main.rs

[workspace]
members = [
    "app",
    "embedding-lib"
]
[dependencies]
embedding-lib = { path = "../embedding-lib" }
```



## impl

给类型添加方法

可以应用于struct，enum，泛型类型

* 使用

```rust
struct User {
    name: String,
    age: u32,
}

impl User {
    // 函数参数不带self
    fn new(name: String, age: u32) -> Self {
        Self { name, age }
    }
	// 只读借用
    fn greet(&self) {
        println!("hello {}", self.name);
    }
    // 修改自身
    fn birthday(&mut self) {
        self.age += 1;
    }
    // 获得所有权
    fn consume(self) {
        println!("{}", self.name);
    }
}

let mut user = User::new("Tom".to_string(), 18);

user.greet();

// 应用于泛型类型
struct Boxed<T> {
    value: T,
}

impl<T> Boxed<T> {
    fn get(&self) -> &T {
        &self.value
    }
}
```



## trait 

类似class，但更关心具体做什么

让不同类型共享同一套行为接口

* 定义trait 

```rust
trait Speak {
    fn speak(&self);
}
```

* 为类型实现 trait

```rust
struct Dog;

impl Speak for Dog {
    fn speak(&self) {
        println!("woof");
    }
}

struct Cat;

impl Speak for Cat {
    fn speak(&self) {
        println!("meow");
    }
}
```

* use

```rust
// 直接使用
let d = Dog;
d.speak();

// 作为函数参数，代表不同的类型有不同的行为-动态写法
fn make_speak(item: &impl Speak) {
    item.speak();
}
// trait bound（约束）
let d = Dog;
make_speak(&d);

// 泛型写法-编译时确定类型，性能高
fn make_speak<T: Speak>(item: &T) {
    item.speak();
}
```
