# 基础

## 获取命令行参数

### 标准库std::env::args()    适合少量参数

~~~
use std::env;
let args: Vec<String> = env::args().collect();  //只能以 String 读，之后做类型转换
//检验参数数目，第一个参数是程序名
if args.len() != 3 {
    eprintln!("用法: <程序名> <i64参数> <字符串参数>");
    return;
}
let number_arg = &args[1];
let string_arg = &args[2];
//类型转换
let number: i64 = match number_arg.parse() {
    Ok(n) => n,
    Err(_) => {
        eprintln!("第一个参数必须是一个 i64 类型的数字！");
        return;
    }
};
~~~

### clap

依赖

`clap = { version = "4", features = ["derive"] }`

```
use clap::Parser;

#[derive(Parser, Debug)]
#[command(author, version, about)]
struct Args {
    /// 第一个参数：i64 类型
    number: i64,
    /// 第二个参数：字符串
    text: String,
}

fn main() {
    let args = Args::parse();
    println!("number = {}, text = '{}'", args.number, args.text);
}
 
```

## main返回

### ()

默认返回单元类型

~~~
fn main(){ }
~~~

### Result<(), Box\<dyn Error>>

~~~
fn main() -> Result<(), Box<dyn std::error::Error>> {
	ok(())
}
~~~

也可以指定错误类型 `Result<(), io::Error>`

## 处理Result<>返回

### 使用 match 语句

match 语句是 Rust 里用于模式匹配的强大工具，能精准地处理 Result 类型的不同变体。

```
let result = get_etf_data();
match result {
    Ok(data) => {
        for etf in data {
            println!("ETF Name: {}, Ticker: {}", etf.name, etf.ticker);
        }
    }
    Err(error) => {
        eprintln!("Error: {}", error);
    }
}
```

在上述代码中，match 语句对 result 进行匹配，若为 Ok 变体，就遍历 Vec\<EtfBaseData> 并打印每条数据；若为 Err 变体，则打印错误信息。

### if let - 只关注一个变体

~~~
let result = get_etf_data();
if let Ok(data) = result {
    for etf in data {
        println!("ETF Name: {}, Ticker: {}", etf.name, etf.ticker);
    }
} else {
    eprintln!("Failed to get ETF data.");
}
~~~

此代码里，if let 语句仅处理 Ok 变体，若为 Ok 则遍历并打印数据，否则打印失败信息。

### unwrap 或 expect

确定肯定没问题的时候使用
遇到错误会直接 panic，expect 可以自定义崩溃时候的错误信息
.expect("必须是数字！");

### ?

出错时会自动返回错误，将错误交给上级，实现链式处理，不会 panic

### map_or_else

先处理错误返回，再处理成功返回

```
check_FullgoaFund_etf(date).map_or_else(
    |e| {
        err_msg = format!("{}\n{}", err_msg, e.to_string());
        "".to_string()
    },
    |s| s
)
```



## 类型转换

转化为 string	 .to_string();

string 转化为 i32    let num = str_num.parse::<i32>().unwrap();

## 判断Option\<T>

### match

```
let maybe_number: Option<i32> = Some(42);
match maybe_number {
    Some(num) => println!("值是: {}", num),  // 处理 Some(T)
    None => println!("没有值"),              // 处理 None
}
```

### if let

```
let maybe_name: Option<String> = Some("Alice".to_string());

if let Some(name) = maybe_name {
    println!("名字是: {}", name);  // 只处理 Some
}

//也可以做到和 match 一样的效果
if let Some(name) = maybe_name {
    println!("名字是: {}", name);
} else {
    println!("没有名字");
}

//如果不关心其值为多少只关心 None 的情况
if let Some(_) = maybe_name {} else {
    println!("没有名字");
}
```

### 方法

```
//return bool
is_some
is_none

if maybe_age.is_some() {
    println!("有年龄值");
}

if maybe_age.is_none() {
    println!("没有年龄值");  // 执行此分支
}
```

### unwrap  expect

```
println!("值是: {}", number.unwrap()); //直接打印值
val_no_value.expect("期望有值!"); //如果没有值直接报错并 panic
```

### unwrap_or  unwrap_or_else

```
name.unwrap_or("Guest".to_string()) //没值则返回一个默认值

count.unwrap_or_else(|| 10 * 2); //没值返回一个闭包计算值
```

### map  and_then

//应用函数

~~~
let maybe_num: Option<i32> = Some(5);
let squared = maybe_num.map(|x| x * x);  // Some(25)

let maybe_str: Option<String> = Some("42".to_string());
let maybe_int = maybe_str.and_then(|s| s.parse::<i32>().ok());  // Some(42)
~~~





# 解析时间

```
fn normalize_date(input: &str) -> Option<String> {
    // 尝试解析为 2025-04-08
    if let Ok(date) = NaiveDate::parse_from_str(input, "%Y-%m-%d") {
        return Some(date.format("%Y%m%d").to_string());
    }

    // 尝试解析为 2025-04-08T00:00:00
    if let Ok(datetime) = NaiveDateTime::parse_from_str(input, "%Y-%m-%dT%H:%M:%S") {
        return Some(datetime.date().format("%Y%m%d").to_string());
    }

    // 尝试解析为 20250409（已是目标格式）
    if input.len() == 8 && input.chars().all(|c| c.is_ascii_digit()) {
        return Some(input.to_string());
    }

    None
}
```



```
static FALLBACK_DATE: OnceCell<String> = OnceCell::new();
FALLBACK_DATE.set(datestr.clone()).unwrap(); 
FALLBACK_DATE.get().cloned().unwrap()
```





### 其他使用

使用 rcp 传输文件



# 库使用

## mysql

```
sqlx = { version = "0.7", features = ["mysql", "runtime-tokio-native-tls", "chrono"] }
use sqlx::{MySqlPool, Row};

const DATABASE_URL: &str = "mysql:://user:password@ip:port/databasename";

let pool = MySqlPool::connect(DATABASE_URL).await?;
    let rows = sqlx::query(
    r#"
    SELECT 
        ...
    WHERE 
    		...
    "#
)
.fetch_all(&pool)
.await?;

for row in rows {
    let trade_date: NaiveDateTime = row.try_get("trade_date")?;
    
    //多行查询结果
    let special_types: Vec<String> = special_rows
        .iter()
        .filter_map(|r| r.try_get("S_TYPE_ST").ok())
        .collect();
    if !special_types.is_empty() {
        record.special_type = Some(special_types.join("_"));//用_拼接
    }
}
```

## std::sync::Lazy

```
use lazy_static::lazy_static;
lazy_static! {
    pub static ref KAFKA_INFO: DashMap<String,HdfsBackupInfo> = DashMap::new();
}
```

在第一次调用KAFKA_INFO 的时候初始化，并且在堆上分配内存，声明周期为全局

更现代和安全的使用：

```
use dashmap::DashMap;
use std::sync::Lazy;

pub static KAFKA_INFO: Lazy<DashMap<String, HdfsBackupInfo>> = Lazy::new(|| {
    DashMap::new()
});
```



# 数据结构

## HashSet

存储唯一元素的集合类型,基于哈希表,增删查找O(1)

~~~rust
use std::collections::HashSet;

//创建
let mut set1: HashSet<i32> = HashSet::new();
// 从一个数组创建 HashSet
let set2: HashSet<i32> = HashSet::from([1, 2, 3]);

//插入
set.insert(3);

//检查元素是否存在
let exists = set.contains(&2);

//删除
let removed = set.remove(&2);

// 并集
let union: HashSet<_> = set1.union(&set2).cloned().collect();
// 交集
let intersection: HashSet<_> = set1.intersection(&set2).cloned().collect();
// 差集
let difference: HashSet<_> = set1.difference(&set2).cloned().collect();

//遍历
for element in &set
~~~

## BTreeSet

use std::collections::BTreeSet;

默认升序排序，元素唯一，插入重复自动忽略，复杂度（O(log n)），多线程需要Mutex



## DashMap

use dashmap::DashMap;

O(1)级别，且线程安全，键唯一

```rust
use dashmap::DashSet;
```



# 特征trait

## 基础使用

不同类型-相同行为

1. 特征定义行为

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

2. 类型实现特征

```rust
pub struct Post {
    pub title: String, // 标题
    pub author: String, // 作者
    pub content: String, // 内容
}

impl Summary for Post {
    fn summarize(&self) -> String {
        format!("文章{}, 作者是{}", self.title, self.author)
    }
}

pub struct Weibo {
    pub username: String,
    pub content: String
}

impl Summary for Weibo {
    fn summarize(&self) -> String {
        format!("{}发表了微博{}", self.username, self.content)
    }
}
```

3. 默认实现，其他类型无需 impl 实现特征,但是其他类型可以重写方法

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

## 使用特征作为函数参数

~~~rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
~~~

这里传入一个实现了特征Summary 的对象

### 特征约束

~~~rsut
pub fn notify<T: Summary>(item1: &T, item2: &T) {}
~~~

约束item1 和item2 拥有相同的类型，且实现约束Summary

多重约束：

`pub fn notify(item: &(impl Summary + Display)) {}`
`pub fn notify<T: Summary + Display>(item: &T) {}`

where约束

~~~rust
fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{}

~~~





多线程

```
    fn start(brokers: String, topics: Vec<String>, seek_from_beginning: bool) {
        std::thread::Builder::new()
            .name(format!("kfk-rdr-{}-{:?}", brokers, topics))
            .spawn(move || Self::consumer(brokers, topics, seek_from_beginning))
            .unwrap();
    }
```

```
git config --global user.name "熊健州"
git config --global user.email "xiongjianzhou@ft.tech"
创建一个新仓库
git clone git@code.non-convex.com:xiongjianzhou/titus_util.git
cd titus_util
git switch -c main
touch README.md
git add README.md
git commit -m "add README"
git push -u origin main
推送现有文件夹
cd existing_folder
git init --initial-branch=main
git remote add origin git@code.non-convex.com:xiongjianzhou/titus_util.git
git add .
git commit -m "Initial commit"
git push -u origin main
推送现有的 Git 仓库
cd existing_repo
git remote rename origin old-origin
git remote add origin git@code.non-convex.com:xiongjianzhou/titus_util.git
git push -u origin --all
git push -u origin --tags
```

# 单元测试

```
#[cfg(test)]
mod tests {
    use data_protocol::models::cb_daily_data_v3::{get_daily_cb_v3, CB_DAILY_TAG};
    #[test]
    fn test_convertible_base_data() {
        let hdfs = hdrs::ClientBuilder::new("hdfs://ftxz-hadoop").connect().unwrap();

        let base_data = get_daily_cb_v3(20211117, Some(&hdfs), CB_DAILY_TAG::V3_1).unwrap();
        //打印其中order_book_id=100001.SH的记录
        let sh_record = base_data.iter().find(|x| x.cb_id == 100001);
        println!("{:?}", sh_record);
    }
}
```

