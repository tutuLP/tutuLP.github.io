# 异步任务超时控制

`tokio::select!` + cancellation token (生产级推荐-优雅)

```rust
let cancel = CancellationToken::new();
let child_cancel = cancel.clone();

let join = tokio::spawn(async move {
    tokio::select! {
        res = do_cleaning() => res,
        _ = child_cancel.cancelled() => Err("canceled".into()),
    }
});

// 外部控制取消
tokio::spawn(async move {
    tokio::time::sleep(timeout).await;
    cancel.cancel();
});

```

## join_handle.abort()

```rust
#[allow(dead_code)]
async fn hello_async() -> Result<String, String> {
    println!("[task] started");
    tokio::time::sleep(Duration::from_secs(4)).await;
    println!("hello");  // ❗ 正常情况 4 秒后才会执行
    Ok("DONE".to_string())
}

/// 你真实业务逻辑的最小可复现版本
#[allow(dead_code)]
async fn execute_timeout_task() -> Result<String, String> {
    let join: tokio::task::JoinHandle<_> = tokio::spawn(async move { hello_async().await });

    pin_mut!(join);

    match timeout(Duration::from_secs(5), &mut join).await {
        Ok(join_result) => match join_result {
            Ok(inner) => inner,                          // 内层 Result
            Err(e) => Err(format!("Join failed: {}", e)),
        },

        Err(_) => {
            println!("[timeout] abort async task!");
            join.abort();
            Err("TIMEOUT".to_string())
        }
    }
}

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_timeout_abort_works() {
    let result = execute_timeout_task().await;
    println!("[result] {:?}", result);

    // ❗ 等待 2 秒，确认 "hello" 不会出现
    println!("(wait 2s to ensure abort worked)");
    tokio::time::sleep(Duration::from_secs(2)).await;

    println!("TEST DONE. If you saw 'hello', abort FAILED.");
}
```

## 可中断sql

IO中断:tokio::io::AsyncRead + cancel-safe wrapper,或者用 tokio 提供的 `CancellationSafe` future

sql驱动需要支持cancel_handle,sqlx是支持的

```rust
use tokio_util::sync::CancellationToken;
use sqlx::Connection;

async fn run_query_with_cancel(token: CancellationToken) -> sqlx::Result<()> {
    let mut conn = pool.acquire().await?;
    let mut handle = conn.handle();

    tokio::select! {
        res = sqlx::query("SELECT SLEEP(20)").execute(&mut conn) => {
            res?;
        }
        _ = token.cancelled() => {
            handle.cancel_query().await?;
            return Err(sqlx::Error::Protocol("Canceled".into()))
        }
    }
    Ok(())
}
```

