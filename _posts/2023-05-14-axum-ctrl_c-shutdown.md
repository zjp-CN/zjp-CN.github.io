---
title: axum 使用 ctrl-c 退出
date: 2023-05-14 15:31:09 +0800
categories: [Rust, test]
tags: [Rust, test, codesnippet]
img_path: /assets/img/2023-05-14-libtest-mimic
---

最简单的例子在 axum 仓库中有：<https://github.com/tokio-rs/axum/blob/b0eb7a24bc62c76d59d2a98117c27a4bdb11a34a/examples/graceful-shutdown/src/main.rs#L31>

但考虑以下场景：使用 ctrl-c 会让 server 结束，而 tokio 异步运行时通过 `block_on` 函数进行的任务可能需要考虑运行时关闭的 [问题]。

```rust
thread 'tokio-runtime-worker' panicked at 'A Tokio 1.x context was found, but it is being shutdown.', ...
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

显然，我们需要告知那些关闭运行时不被处理的任务在关闭前结束。

告知的方式有很多种，比如 channel。见 tokio 博文 [Graceful Shutdown](https://tokio.rs/tokio/topics/shutdown)。

核心技巧是通过 `select!` 让异步任务在长时间情况下结束，并 await 这个任务。

以下的关键代码是涉及 `notify` 几行和 `blocking_task.await` 一行。[^src]

```rust
/*
[dependencies]
axum = "0.6.18"
tokio = { version = "1.28.0", features = ["full"] }
*/

use axum::{Router, Server};
use std::{net::SocketAddr, sync::Arc, time::Duration};
use tokio::{runtime::Handle, select, sync::Notify, time::sleep};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the socket & the router
    let socket = SocketAddr::new("0.0.0.0".parse()?, 12678);
    let router = Router::new();
    /* many routes */

    // notify the event of ctrl-c: can be given to multiple tasks
    let notify = Arc::new(Notify::new());

    // Start the cache manager thread
    let blocking_task = {
        let notify = notify.clone();
        Handle::current().spawn_blocking(move || {
            Handle::current().block_on(async move {
                // ...

                // finished when the sleep task is done or the ctrl-c signal is notified
                select! {
                    _ = sleep(Duration::from_secs(3)) => println!("sleep done"),
                    _ = notify.notified() => println!("server is shutting down"),
                }
            });
        })
    };

    // Start the Axum HTTP server
    Server::bind(&socket)
        .serve(router.into_make_service())
        .with_graceful_shutdown({
            let notify = notify.clone();
            async move {
                tokio::signal::ctrl_c()
                    .await
                    .expect("Failed to catch the SIGINT signal");
                notify.notify_waiters();
            }
        })
        .await?;

    // ensure the blocking_task is finished before runtime shuts down
    notify.notify_waiters(); // in case ctrl_c finishes before blocking_task
    blocking_task.await?;

    Ok(())
}
```

[^src]: 例子来自于我回答的 <https://users.rust-lang.org/t/tokio-runtime-panics-at-shutdown/93651>
