

# Mysql

```toml
anyhow = "1.0"
sqlx = { version = "0.7", features = ["mysql", "runtime-tokio-native-tls", "chrono", "rust_decimal"] }
```

## Mysql 连接池

```rust
use anyhow::Result;
use sqlx::{mysql::MySqlPoolOptions, MySql, Pool};
use std::time::Duration;
use serde::Deserialize;

#[derive(Debug, Deserialize, Clone)]
pub struct MySqlConfig {
    pub url: String,
    pub max_connections: u32,   // 最大连接数: 20 (支持多程序并发)
    pub acquire_timeout: u64,   // 连接超时: 30秒
    pub idle_timeout: u64,      // 空闲超时: 10分钟
    pub max_lifetime: u64,      // 最大生命周期: 30分钟
}

/// 创建 MySQL 连接池
///
/// # 参数
/// * `mysql_config` - MySQL 配置
/// ```
pub async fn create_mysql_pool(
    mysql_config: &MySqlConfig
) -> Result<Pool<MySql>> {
    let pool = MySqlPoolOptions::new()
        .max_connections(mysql_config.max_connections)
        .acquire_timeout(Duration::from_secs(mysql_config.acquire_timeout))
        .idle_timeout(Duration::from_secs(mysql_config.idle_timeout))
        .max_lifetime(Duration::from_secs(mysql_config.max_lifetime))
        //.test_before_acquire(true) // 获取连接前测试，DB经常断连的环境使用
        // 每个连接创建时自动设置时区
        .after_connect(|conn, _meta| {
                Box::pin(async move {
                sqlx::query("SET time_zone = '+08:00'")
                        .execute(conn)
                        .await?;
                    Ok(())
                })
        })
        .connect(&mysql_config.url)
        .await?;

    Ok(pool)
}
```

## 一次性连接

```rust
use sqlx::{Connection, MySqlConnection};

/// 创建一次性 MySQL 连接
///
/// # 使用场景
/// - CLI 工具
/// - 一次性脚本
/// - 事务
///
/// # 注意
/// - 不要在 Web / 长期运行服务中使用
pub async fn create_mysql_connection(mysql_url: &str) -> Result<MySqlConnection> {
    let mut conn = MySqlConnection::connect(mysql_url).await?;

    // 可选：设置会话级参数
    sqlx::query("SET time_zone = '+08:00'")
        .execute(&mut conn)
        .await?;

    Ok(conn)
}
```

## 使用

#### create_mysql_pool

* AppState 用结构体管理连接

web：axum::Extension 使用
多线程：app_state.clone() 直接clone使用，但是注意不要直接在std::thread中使用，可以在Tokio中使用，std::thread更适合一次性连接

```rust
#[derive(Clone)]
struct AppState {
    mysql: Pool<MySql>,
}

// 初始化 MySQL 连接池 允许使用unwrap，创建错误则退出
let mysql_pool = create_mysql_pool(&config.mysql).await.unwrap_or_else(|err| {
    eprintln!("错误：无法创建 MySQL 连接池");
    eprintln!("详细信息: {}", err);
    std::process::exit(1);
});

// 初始化全局资源
let app_state = AppState {
    mysql: mysql_pool,
};
```

* OnceCell 不建议，但是也能用，简单方便

```rust
static MYSQL_POOL: OnceCell<Pool<MySql>> = OnceCell::const_new();

pub async fn mysql_pool() -> &'static Pool<MySql> {
    MYSQL_POOL
        .get_or_init(|| async {
            create_mysql_pool(MYSQL_URL).await
                .expect("mysql pool init failed")
        })
        .await
}
```

#### create_mysql_connection
适合CLI，脚本等
适合事务使用
使用std::thread

# PG

```toml
[postgres]
url = "postgres://user:password@127.0.0.1:5432/quote"
max_connections = 16
acquire_timeout = 30
idle_timeout = 600
max_lifetime = 1800
statement_timeout = 30
```



```rust
use anyhow::Result;
use serde::Deserialize;
use sqlx::{postgres::PgPoolOptions, Pool, Postgres};
use std::time::Duration;

#[derive(Debug, Deserialize, Clone)]
pub struct PostgresConfig {
    pub url: String,
    pub max_connections: u32,           // 最大连接数
    pub acquire_timeout: u64,           // 获取连接超时（秒）
    pub idle_timeout: u64,              // 空闲超时（秒）
    pub max_lifetime: u64,              // 最大生命周期（秒）
    pub statement_timeout: Option<u64>, //单条 SQL 最大执行时间（秒）
}

/// 创建 PostgreSQL 连接池
pub async fn create_postgres_pool(pg_config: &PostgresConfig) -> Result<Pool<Postgres>> {
    let statement_timeout = pg_config.statement_timeout;

    let pool = PgPoolOptions::new()
        .max_connections(pg_config.max_connections)
        .acquire_timeout(Duration::from_secs(pg_config.acquire_timeout))
        .idle_timeout(Duration::from_secs(pg_config.idle_timeout))
        .max_lifetime(Duration::from_secs(pg_config.max_lifetime))
        //.test_before_acquire(true) // PG 经常重启 / 网络不稳时可打开
        .after_connect(move |conn, _meta| {
            Box::pin(async move {
                // 设置时区（PG 推荐使用 SET TIME ZONE）
                sqlx::query("SET TIME ZONE 'Asia/Shanghai'")
                    .execute(&mut *conn)
                    .await?;

                // 限制单条 SQL 最大执行时间（防止慢查询拖垮池）
                if let Some(seconds) = statement_timeout {
                    let sql = format!("SET statement_timeout = '{}s'", seconds);
                    sqlx::query(&sql).execute(&mut *conn).await?;
                }

                Ok(())
            })
        })
        .connect(&pg_config.url)
        .await?;

    Ok(pool)
}

```



# Redis

返回redis连接池，封装队列的操作方法，并且一般只有长服务呀，异步，并发，web等才使用并且也是需要redis_pool的场景所以就不提供获取单连接的方法了

使用：同样建议放在AppState结构体中

```toml
redis = { version = "0.32", features = ["tokio-comp"] }
deadpool-redis = { version = "0.22.0", features = ["rt_tokio_1"] }
```

```toml
[redis]
url = "redis://:password@ft11:6379/0"
max_size = 16
wait_timeout = 3
create_timeout = 3
recycle_timeout = 3
```

```rust
use std::time::Duration;

use anyhow::{Context, Result};
use deadpool_redis::{Config, Pool, PoolConfig, Runtime, Timeouts};
use serde::Deserialize;


#[derive(Debug, Deserialize, Clone)]
pub struct RedisConfig {
    pub url: String,
    pub max_size: usize,        // 最大连接数
    pub wait_timeout: u64,      // 池满后等待连接可用时间 s
    pub create_timeout: u64,    // 新建连接超时时间 s
    pub recycle_timeout: u64,   // 检测链接可用最大时间 s
}


/// 创建 Redis 连接池
///
/// # 参数
/// * `redis_config` - Redis 配置
///
/// # 返回
/// * Ok(Pool) - 创建成功的连接池
/// * Err(anyhow::Error) - 创建失败原因
pub fn create_redis_pool(redis_config: &RedisConfig) -> Result<Pool> {
    let cfg = Config {
        url: Some(redis_config.url.clone()),
        pool: Some(PoolConfig {
            max_size: redis_config.max_size,
            timeouts: Timeouts {
                wait: Some(Duration::from_secs(redis_config.wait_timeout)),
                create: Some(Duration::from_secs(redis_config.create_timeout)),
                recycle: Some(Duration::from_secs(redis_config.recycle_timeout)),
            },
            ..Default::default()
        }),
        ..Default::default()
    };

    cfg.create_pool(Some(Runtime::Tokio1))
        .context("failed to create redis connection pool")
}


use redis::AsyncCommands;


/// Redis 队列操作仓库（基于 List）
///
/// # 说明
/// - 内部使用 deadpool-redis 连接池
/// - 每次方法调用都会从池中获取短生命周期连接
/// - 适合用于任务队列 / 消息队列 / 异步消费场景
pub struct QueueRepo {
    pool: Pool,
}

impl QueueRepo {
    /// 创建新的队列仓库实例
    ///
    /// # 参数
    /// * `pool` - Redis 连接池（建议在 AppState 中统一管理）
    pub fn new(pool: Pool) -> Self {
        Self { pool }
    }

    /// 向队列右侧推入一个元素（RPUSH）
    ///
    /// # 参数
    /// * `key` - Redis 列表 key
    /// * `value` - 推入的字符串值
    ///
    /// # 行为
    /// - 成功时返回 Ok(())
    /// - 失败时返回 anyhow::Error，包含上下文信息
    pub async fn push(&self, key: &str, value: &str) -> Result<()> {
        let mut conn = self
            .pool
            .get()
            .await
            .context("failed to get redis connection from pool")?;

        conn.rpush::<_, _, ()>(key, value)
            .await
            .with_context(|| format!("failed to rpush value to redis list: key={}", key))?;

        Ok(())
    }

    /// 阻塞式从队列左侧弹出一个元素（BLPOP）
    ///
    /// # 参数
    /// * `key` - Redis 列表 key
    /// * `timeout` - 阻塞超时时间（秒）
    ///     - 0 表示永久阻塞
    ///
    /// # 返回
    /// - Ok(Some(value))：成功弹出一个元素
    /// - Ok(None)：超时未获取到元素
    /// - Err(anyhow::Error)：Redis 或连接错误
    pub async fn pop_blocking(
        &self,
        key: &str,
        timeout: usize,
    ) -> Result<Option<String>> {
        let mut conn = self
            .pool
            .get()
            .await
            .context("failed to get redis connection from pool")?;

        let res: Option<(String, String)> = redis::cmd("BLPOP")
            .arg(key)
            .arg(timeout)
            .query_async(&mut conn)
            .await
            .with_context(|| format!("failed to blpop from redis list: key={}", key))?;

        Ok(res.map(|(_, v)| v))
    }
}
```

# Ftlog

```toml
ftlog = "0.2.12"
```



日志级别：Debug Info Warn Error

```rust
use ftlog::{appender::{Duration, Period}, LevelFilter};

pub fn log_init(log_path: &str) {
    ftlog::builder()
        .max_log_level(LevelFilter::Info)// 设置日志级别
        // .filter("axum_keycloak_auth", None, LevelFilter::Warn) // 过滤日志
        .unbounded()
        .root(ftlog::appender::FileAppender::rotate_with_expire(
            log_path,
            Period::Day,
            Duration::days(5), // 日志保留天数
        ))
        .build()
        // 日志系统初始化失败应该panic
        .expect("logger build failed")
        .init()
        // 日志系统设置失败应该panic
        .expect("set logger failed ");
    // 5秒延迟：等待日志系统完全初始化，此延迟不影响性能因为只在程序启动时执行一次
    std::thread::sleep(core::time::Duration::from_secs(5));
}
```

## 使用

标准路径参考：
`/data2/share/logalert/FT_BASEDATA_CHECK_TEST#xiongjianzhou_index_weight_mysql_xz/app.log`

```rust
use ftlog::{error, info, logger};

log_init
info!("{}", str);

// 保证日志写入完成
// 注意：此处的sleep(3秒)不会导致性能问题，因为：
// 1. 这是程序结束前的清理操作，不在热路径上
// 2. 3秒的等待时间确保异步日志系统完全刷新缓冲区
logger().flush();
thread::sleep(std::time::Duration::from_secs(3));
```

# 全局消息实现

## 多线程版本

```toml
once_cell = "1.19"
parking_lot = "0.12"
```

```rust
use once_cell::sync::Lazy;
use parking_lot::Mutex;

/// 全局多线程消息缓冲
static GLOBAL_MESSAGE: Lazy<Mutex<String>> =
    Lazy::new(|| Mutex::new(String::new()));

/// 追加消息（尾添）
pub fn append_message(msg: impl AsRef<str>) {
    let mut buf = GLOBAL_MESSAGE.lock();
    if !buf.is_empty() {
        buf.push('\n');
    }
    buf.push_str(msg.as_ref());
}

/// 获取全部消息
pub fn get_message() -> String {
    GLOBAL_MESSAGE.lock().clone()
}

/// 清空消息
pub fn clear_message() {
    GLOBAL_MESSAGE.lock().clear();
}

/// 是否为空
pub fn is_empty() -> bool {
    GLOBAL_MESSAGE.lock().is_empty()
}

/// 消息长度
pub fn len() -> usize {
    GLOBAL_MESSAGE.lock().len()
}

```

## 单文件使用

thread_local：一个“只能通过共享引用 &T 访问”的线程本地静态变量

RefCell：确定安全的情况使用，无锁，单线程使用，性能高

```rust
use std::cell::RefCell;

thread_local! {
    static MESSAGE: RefCell<String> = RefCell::new(String::new());
}

/// 追加消息
pub fn append_message(msg: impl AsRef<str>) {
    MESSAGE.with(|m| {
        let mut buf = m.borrow_mut();
        if !buf.is_empty() {
            buf.push('\n');
        }
        buf.push_str(msg.as_ref());
    });
}

/// 获取消息
pub fn get_message() -> String {
    MESSAGE.with(|m| m.borrow().clone())
}

/// 清空消息
pub fn clear_message() {
    MESSAGE.with(|m| m.borrow_mut().clear());
}

/// 是否为空
pub fn is_empty() -> bool {
    MESSAGE.with(|m| m.borrow().is_empty())
}

/// 消息长度
pub fn len() -> usize {
    MESSAGE.with(|m| m.borrow().len())
}

```

## 使用

两者使用相同

```
append_message("step 1 ok");
append_message("step 2 ok");

println!("{}", get_message());

clear_message();
```

# 项目依赖检查

```shell
# 下载 cargo-udeps
cargo install cargo-udeps
# rustup toolchain install nightly # 安装失败可能需要使用

# 使用
cargo +nightly udeps
cargo +nightly udeps --workspace --all-targets
```

# Tokio Runtime Handle

全局Tokio Runtime Handle应用场景：

* 同步代码或普通线程中调用异步函数
* 已有Runtime不能再new runtime
* 不适合在async函数中使用
* 不适合web handler，只适合同步普通线程
* 不适合一次性脚本等非多线程环境
* 可以多个线程同时使用，但是一个线程只能一个runtime

```rust
use anyhow::{anyhow, Result};
use once_cell::sync::OnceCell;
use tokio::runtime::Handle;

/// 全局 Tokio Runtime Handle
/// 用于在普通线程中执行异步任务
static RUNTIME_HANDLE: OnceCell<Handle> = OnceCell::new();

/// 初始化全局 runtime handle
/// 应该在 tokio runtime 内部调用（例如在 #[tokio::main] 标记的函数中）
pub fn init_runtime_handle() -> Result<()> {
    let handle = Handle::try_current()
        .map_err(|_| anyhow!("not running inside a Tokio runtime"))?;

    RUNTIME_HANDLE
        .set(handle)
        .map_err(|_| anyhow!("runtime handle already initialized"))?;

    Ok(())
}

/// 获取全局 runtime handle
///
/// # Panics
/// 如果 runtime 未初始化会 panic
pub fn get_runtime_handle() -> Result<&'static Handle> {
    RUNTIME_HANDLE
        .get()
        .ok_or_else(|| anyhow!("Runtime handle not initialized. Call init_runtime_handle() first."))
}
/// 在全局 runtime 中执行异步任务（阻塞当前线程直到完成）
///
/// # 示例
/// ```no_run
/// use crate::utils::runtime::block_on;
///
/// let result = block_on(async {
///     // 异步代码
///     "result"
/// });
/// ```
#[allow(dead_code)]
pub fn block_on<F>(future: F) -> anyhow::Result<F::Output>
where
    F: std::future::Future + Send,
    F::Output: Send,
{
    Ok(get_runtime_handle()?.block_on(future))
}

/// 在全局 runtime 中生成一个异步任务
/// 返回 JoinHandle，可以用来等待或取消任务
///
/// # 示例
/// ```no_run
/// use crate::utils::runtime::spawn;
///
/// let handle = spawn(async {
///     // 异步代码
/// });
///
/// // 可以等待任务完成
/// // handle.await
///
/// // 或者取消任务
/// // handle.abort()
/// ```
#[allow(dead_code)]
pub fn spawn<F>(future: F) -> Result<tokio::task::JoinHandle<F::Output>>
where
    F: std::future::Future + Send + 'static,
    F::Output: Send + 'static,
{
    Ok(get_runtime_handle()?.spawn(future))
}
```

