> 原文链接: https://kerkour.com/rust-async-await-what-is-a-runtime
>
> **翻译：[trdthg](https://github.com/trdthg)**
>
> 选题：[trdthg](https://github.com/trdthg)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 异步 Rust：什么是运行时？下面是 tokio 的工作原理

> 这部分可以参考
> [异步运行时及 tokio 概览](https://course.rs/async-rust/tokio/overview.html#tokio-%E6%A6%82%E8%A7%88)

Rust 本身只提供了异步编程所需的基本特性，例如 `async/.await` 关键字，`Future` 特征等。但是并不提供执行 `Futures` 和
`Streams` 执行所需的上下文环境。这个执行上下文被称为运行时。如果没有运行时，你就不能运行异步代码。

目前最受欢迎的几个运行时有：

| 运行时       | 总下载量 (2022-01) | 简介                      |
| --------- | -------------- | ----------------------- |
| tokio     | 42,874,882     | 提供了一个事件驱动型的非阻塞 I/O 平台   |
| async-std | 5,875,271      | Rust 标准库的异步版本，跟标准库兼容性较强 |
| smol      | 1,187,600      | 一个小而快的异步运行时             |

各个运行时之间并不兼容，你不能通过仅靠更改一两行代码就将运行时切换为另一个。

不同运行时的生态系统也是支离破碎的。你必须选择一个运行时并在未来坚持使用下去。

## Tokio 简介

Tokio 是 Rust 异步运行时，社区支持最强大，并且有许多赞助商 (例如 Discord、Fly.io 和 Embark)，所以它有很多
[付费贡献者](https://github.com/sponsors/tokio-rs#sponsors) 如果你不是在进行嵌入式开发，那么 Tokio
就是你的不二之选。

## 事件循环

所有异步运行时（无论是 Rust、Node.js 还是其他语言）的核心都是 **事件循环 (event loops)**，或者称为 **处理器
(processors)**。
![](https://kerkour.com/2022/rust-async-await-what-is-a-runtime/ch03_event_loop.png)

<div style="text-align:center;"> Work stealing runtime. By Carl Lerche - License MIT -
https://tokio.rs/blog/2019-10-scheduler#the-next-generation-tokio-scheduler
</div>
在实践中，为了拥有更好的性能，每个程序会拥有不止一个执行器，每一个 CPU 核心都会有一个自己的执行器。

每个执行器都有自己的任务队列。但是如果某一个执行器的队列是空的，它就会尝试从其他执行器的任务队列里 **"偷" 任务**。

> 要了解有关不同类型事件循环的更多信息，你可以阅读 Carl Lerche
> 的这篇优秀文章：[Making the Tokio scheduler 10x faster](https://tokio.rs/blog/2019-10-scheduler)。

## 分配任务

你可以通过 `tokio` 的 `tokio::spawn` 函数把任务分派给运行时。

```rs
tokio::spawn(async move {
    for port in MOST_COMMON_PORTS_100 {
        let _ = input_tx.send(*port).await;
    }
});
```

此代码片段会生成一个任务，并把它推入其中一个执行器的队列中。由于每个执行器都有自己的操作系统线程，所以 Tokio
会帮我们调度机器的所有资源，而无需自己亲自管理线程。如果你没有使用 `spawn`，所有任务都会分配到同一个处理器上，并且在同一个线程上执行。

作为比较，在 Go 中，我们使用 go 关键字来生成一个任务（称为 goroutine）：

```go
go doSomething()
```

## 避免事件循环被阻塞

**接下来是本文最重要的部分**

在 `async-await` 的世界中要记住，千万不要让事件循环阻塞。

意思就是不要直接调用可能运行超过
[10 到 100 微秒](https://ryhl.io/blog/async-what-is-blocking/)的函数。你应该将这些任务放到单独的阻塞队列里。

**CPU 密集型操作**

当我们遇到计算密集型操作，例如加密、图像编码或文件散列时应该怎么做？

Tokio 为这种可能引起阻塞的操作提供了
[`tokio::task::spaen_blocking`](https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html)
函数。阻塞操作指的是一个不会无限运行的后台任务。对于这种任务，给它分配一个单独的 Rust 线程更合适。

下面是使用 `spawn_blocking` 的示例：

```rs
let is_code_valid = spawn_blocking(move || crypto::verify_password(&code, &code_hash)).await?;
```

函数 `crypto::verify_password` 预计需要数百毫秒才能完成，它会阻塞事件循环。通过调用 `spawn_blocking`，它将被被分派到
`tokio` 的阻塞任务线程池。

![](https://kerkour.com/2022/rust-async-await-what-is-a-runtime/ch03_tokio_threadpools.png)

Tokio 在底层维护着两个线程池。

一个是固定大小的线程池，分配给执行异步任务的执行器（事件循环、处理器）。可以使用 `tokio::spawn` 将异步任务分派到此线程池

还有一个动态大小的线程池，分配给阻塞任务使用。

默认情况下，后者将增长到 512 个线程，它的大小会根据正在运行的阻塞任务数量而扩大或缩小。可以使用
`tokio::task::spawn_blocking`将阻塞任务分派到此线程池。你可以在
[tokio 的文档](https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.max_blocking_threads)
中阅读更多信息。

这就是为什么 `async-await` 也被称为 "绿色线程" 或 "M:N 线程"。一个 Tokio
任务是一个异步的绿色线程，它看起来像用户线程，但生成的开销更小，而且你可以创建比操作系统线程更多的绿色线程供运行时使用。

[异步与同步共存](https://course.rs/async-rust/tokio/bridging-with-sync.html)
