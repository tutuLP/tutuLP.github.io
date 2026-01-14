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



## 编辑器

* clion

rust插件：https://plugins.jetbrains.com/plugin/8182--deprecated-rust

* vscode

插件：rust-analyzer

# start

main.rs

~~~rust
fn main() {
    println!("Hello, world!");
}
~~~

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

~~~~rust
struct UnPrintable(i32); //不能直接使用`fmt::Display` 或 `fmt::Debug` 来进行打印

#[derive(Debug)] 
struct DebugPrintable(i32);//`derive` 属性会自动创建所需的实现，使这个 `struct` 能使用 `fmt::Debug` 打印
~~~~

用{:?}打印
使用和{}一样：{1:?} {0:?} {actor:?}

美化打印 {:#?}

~~~rust
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
~~~

### 显示Display

display不像debug声明#[derive(Debug)]后就会自动提供实现，display必须手动为类型实现 fmt::Display trait，display更简洁，导致Vec<T> 或其他任意泛型容器用用 fmt::Debug 而不用Display，因为输出样式不同

~~~rust
use std::fmt;  //导入模块使`fmt::Display` 可用
struct Structure(i32);  
impl fmt::Display for Structure {  //impl：为 Structure 结构体实现 fmt::Display trait
    // 这个 trait 要求 `fmt` 使用与下面的函数完全一致的函数签名
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        // 仅将 self 的第一个元素写入到给定的输出流 `f`。返回 `fmt:Result`，此结果表明操作成功或失败。注意 `write!` 的用法和 `println!` 很相似。
        write!(f, "{}", self.0)
    }
}
~~~

对比Display和debug

~~~rust
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
~~~

### 测试实例List

write!配合?操作符实现复杂的拼接，因为使用write都会生成一个Resualt，这里使用?就如果成功就继续，是被则从fmt返回错误，减少了if的判断

~~~rust
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
~~~

### 格式化

~~~
- `format!("{}", foo)` -> `"3735928559"`
- `format!("0x{:X}", foo)` -> [`"0xDEADBEEF"`](https://en.wikipedia.org/wiki/Deadbeef#Magic_debug_values)
- `format!("0o{:o}", foo)` -> `"0o33653337357"`
~~~

~~~rust
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
~~~

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

~~~rust
let logical: bool = true; //常规说明
let an_integer   = 5i32; //后缀说明
let default_float   = 3.0;//默认
let mut mutable = 12;//可变值，但是变量的类型不能改变
~~~

## 字面量和运算符

通过加前缀 `0x`、`0o`、`0b`，数字可以用十六进制、八进制或二进制记法表示。
可以在数值字面量中插入下划线，比如：`1_000` 等同于 `1000`，`0.000_001` 等同于 `0.000001`。

~~~rust
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
~~~

## 元组

元组是一个可以包含各种类型值的组合，可以拥有任意个值

~~~rust
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
~~~

## 数组和切片

数组：[T; length] 类型和大小，相同类型，连续存储
切片：和数组类似，但是大小在编译时不确定，双字对象，指向数据的指针+切片长度，字大小是usize，处理器架构决定

slice 可以用来借用数组的一部分。slice 的类型标记为 `&[T]`。

~~~rust
use std::mem;//获取字节数
// 此函数借用一个 slice
fn analyze_slice(slice: &[i32]) {
    println!("first element of the slice: {}", slice[0]);
    println!("the slice has {} elements", slice.len());
}

fn main() {
    // 定长数组（类型标记是多余的）
    let xs: [i32; 5] = [1, 2, 3, 4, 5];
    // 所有元素可以初始化成相同的值
    let ys: [i32; 500] = [0; 500];
    // 下标从 0 开始
    println!("first element of the array: {}", xs[0]);

    // `len` 返回数组的大小
    println!("array size: {}", xs.len());
    // 数组是在栈中分配的
    println!("array occupies {} bytes", mem::size_of_val(&xs));

    // 数组可以自动被借用成为 slice
    println!("borrow the whole array as a slice");
    analyze_slice(&xs);

    // slice 可以指向数组的一部分
    println!("borrow a section of the array as a slice");
    analyze_slice(&ys[1 .. 4]);

    // 越界的下标会引发致命错误（panic）
    //println!("{}", xs[5]);
}

~~~



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

### return Err

手动返回错误

```rust
if config.redis_url.is_empty() {
    return Err(anyhow!("redis_url is empty"));
}
```



# warn

* the following packages contain code that will be rejected by a future version of Rust: redis v0.25.4

升级依赖redis的版本

* warning: `/Users/xzmini11/.cargo/config` is deprecated in favor of `config.toml`

mv ~/.cargo/config ~/.cargo/config.toml
