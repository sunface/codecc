> 原文链接: https://www.zupzup.org/epoll-with-rust/index.html
>
> **翻译：[BK0717](https://github.com/hyuuko)**
>
> 选题：[trdthg](https://github.com/trdthg)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# 在 Rust 中使用 epoll 实现基本的非阻塞 IO

我已经在许多语言中（比如 JavaScript(Node.js)、Java/Kotlin 和 Rust）使用异步或非阻塞 IO 许多年了。

然而，我无法很好地解释 `poll`、`select`、`epoll`、`kqueue` 的工作原理，不是很有信心解释清楚非阻塞 IO 实际上是如何工作的。

在这篇文章里我们会用 Rust 实现一个非常简单的非阻塞 HTTP 服务器，它会仅使用一个线程来同时为多个请求提供服务。

这个例子会在 Linux 上运行，因此我们会使用 `epoll` 系列系统调用来达成非阻塞 IO 的目标。

在本文末尾的参考资料部分中，你将找到用于编写此内容的一些教程和库的链接。这些参考资料对 Linux，Windows 和 MacOS 都有用，如果您对其他平台感兴趣或者在更深入地理解主题，我强烈建议您看一下。

## 初步设置

我们唯一要使用的依赖是 libc，并且实际上我们不会处理错误。我们将会把所有的错误传播到顶部以保持简单，如果有任何问题，程序将 panic 并且停止。

首先，让我们定义一个从 mio 复制过来的辅助用的宏，用来调用 libc 的系统调用并且用 `Result` 包装结果。

```rust
#[allow(unused_macros)]
macro_rules! syscall {
    ($fn: ident ( $($arg: expr),* $(,)* ) ) => {{
        let res = unsafe { libc::$fn($($arg, )*) };
        if res == -1 {
            Err(std::io::Error::last_os_error())
        } else {
            Ok(res)
        }
    }};
}
```

这个宏没什么好讲的，只是在一个 unsafe 块中调用给定的 libc 系统调用，如果它返回一个错误(-1)，错误被包装在 `Result` 中，反之返回一个用 Ok 包装的值。

接下来，我们需要在 `127.0.0.1:8000` 启动一个 TCP 服务器，我们可以稍候进行测试：

```rust
use std::os::unix::io::{AsRawFd, RawFd};
use std::io;
use std::io::prelude::*;
use std::net::{TcpListener, TcpStream};

const HTTP_RESP: &[u8] = b"HTTP/1.1 200 OK
content-type: text/html
content-length: 5

Hello";

fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8000")?;
    listener.set_nonblocking(true)?;
    let listener_fd = listener.as_raw_fd();

    ...

    Ok(())
}
```

这段代码在给定的套接字地址上创建了一个 `TcpListener`。我们将它设置成了非阻塞模式，这意味着使我们能够接受新的连接的 `accept` 操作变成了非阻塞的。如果此时 socket 还没准备好而我们立即 `accept`，`accept` 将不会阻塞，而是直接返回一个 `io::ErrorKind::WouldBlock` 错误告诉我们稍后再重新尝试。

最后，记住这个文件描述符（在 Linux 上只是一个整数），我们稍后会用到它。我们还定义了一个常量 `HTTP_RESP`，这是我们对所有传入 HTTP 请求的响应。

接下来，让我们设置我们的 epoll 事件队列。

```rust
fn main() -> io::Result<()> {
    ...

    let epoll_fd = epoll_create().expect("can create epoll queue");
    ...
}

fn epoll_create() -> io::Result<RawFd> {
    let fd = syscall!(epoll_create1(0))?;
    if let Ok(flags) = syscall!(fcntl(fd, libc::F_GETFD)) {
        let _ = syscall!(fcntl(fd, libc::F_SETFD, flags | libc::FD_CLOEXEC));
    }

    Ok(fd)
}
```

`epoll_create` 简单地调用了 [epoll_create][epoll_create] 系统调用，并返回用于 epoll 事件通知的文件描述符。

通过这个文件描述符，我们可以添加、修改和移除对特定文件描述符上的特定事件的兴趣，当我们稍后调用 [epoll_wait][epoll_wait] 时，它将阻塞直到相应事件发生然后通知我们，我们就可以在给定的文件描述符上进行读取（比如一个传入的连接）。

一步一步来，现在我们已经有了 epoll，我们想要添加一个对传入的连接的兴趣。

```rust
const READ_FLAGS: i32 = libc::EPOLLONESHOT | libc::EPOLLIN;
const WRITE_FLAGS: i32 = libc::EPOLLONESHOT | libc::EPOLLOUT;

fn main() -> io::Result<()> {
    ...
    let mut key = 100;
    add_interest(epoll_fd, listener_fd, listener_read_event(key))?;
    ...
}

fn add_interest(epoll_fd: RawFd, fd: RawFd, mut event: libc::epoll_event) -> io::Result<()> {
    syscall!(epoll_ctl(epoll_fd, libc::EPOLL_CTL_ADD, fd, &mut event))?;
    Ok(())
}

fn listener_read_event(key: u64) -> libc::epoll_event {
    libc::epoll_event {
        events: READ_FLAGS as u32,
        u64: key,
    }
}
```

`add_interest` 函数的参数有：`epoll` 实例的文件描述符、我们想要添加兴趣的文件描述符，以及一个定义了我们想要被通知的事件类型的 `epoll_event`。

这个事件有两个元素，`events` 和 `u64`。`events` 是一组标志，根据我们感兴趣的事件而设置。在此例中，我们只对读就绪事件感兴趣，因此用 `EPOLLIN`。然而，在读写的情况下，我们额外设置了 `EPOLLONESHOT` 标志，这意味着当我们根据此兴趣得到通知时，该兴趣将被移除。

因此，如果我们想要保持读取（因为我们期待这里有更多的数据），我们需要再次注册该兴趣。

这些参数通过使用 `EPOLL_CTL_ADD` 标志的 [epoll_ctl][epoll_ctl] 被传递，表明我们正在添加一个兴趣。

我们的服务器已经建立起来了，epoll 队列和对新的连接的兴趣也有了，下一步就是我们的事件循环！

```rust
fn main() -> io::Result<()> {
    ...
    let mut events: Vec<libc::epoll_event> = Vec::with_capacity(1024);
    ...
    loop {
        events.clear();
        let res = match syscall!(epoll_wait(
            epoll_fd,
            events.as_mut_ptr() as *mut libc::epoll_event,
            1024,
            1000 as libc::c_int,
        )) {
            Ok(v) => v,
            Err(e) => panic!("error during epoll wait: {}", e),
        };

        // safe  as long as the kernel does nothing wrong - copied from mio
        unsafe { events.set_len(res as usize) };
        ...
    }
}
```

我们启动了一个无限循环。在这个循环里，我们做的第一件事就是清除我们的 event vector。这个 `events` vector 大小为 1024，我们将会把它传递给 `epoll_wait`，也就是设置新的事件的地方。

这也是为什么我们清除它，因为我们不想要任何的旧事件存在。

然后我们用前面提到的 `epoll_fd` 调用 `epoll_wait`，事件数量的最大值应在以毫秒为单位的 timeout 内被返回。如果没有事件发生，将会到达此 timeout。

一旦调用了，`epoll_wait` 将会阻塞直到以下三件事情发生：

- 我们注册了兴趣的文件描述符有一个事件
- 一个信号处理函数打断了该系统调用
- timeout 过期

我们可以将 timeout 设置成 -1，这会让 `epoll_wait` 一直阻塞，直到任意其他两件事情之一发生。

待 `epoll_wait` 返回后，我们就得到了一个表示事件数量的 `res` 变量。然后我们设置 `events` 的长度。这里我们要使用 `unsafe`，因为 `set_len` 不会为我们做任何安全检查。

通常我们需要确保 `res` 小于等于 `events` 的容量，并且在旧长度内的值（在该例中为 0）在新长度的情况下不会变化。这里是安全的，因为它返回的值实际上是系统调用设置的。这段代码是我从 mio 复制过来的，在其他教程中也见过，所以应该没问题。

接下来，让我们看看我们取到的事件并开始接受和处理请求！

## 处理请求

首先，我们简单的迭代 `events` 并匹配 `events` 里的 u64 值，这是我们对感兴趣的每个文件描述符提供的唯一的 key。

```rust
#[derive(Debug)]
pub struct RequestContext {
    pub stream: TcpStream,
    pub content_length: usize,
    pub buf: Vec<u8>,
}

impl RequestContext {
    fn new(stream: TcpStream) -> Self {
        Self {
            stream,
            buf: Vec::new(),
            content_length: 0,
        }
    }
}

fn main() -> io::Result<()> {
    ...
    let mut request_contexts: HashMap<u64, RequestContext> = HashMap::new();
    ...
        println!("requests in flight: {}", request_contexts.len());
        for ev in &events {
            match ev.u64 {
                100 => {
                    match listener.accept() {
                        Ok((stream, addr)) => {
                            stream.set_nonblocking(true)?;
                            println!("new client: {}", addr);
                            key += 1;
                            add_interest(epoll_fd, stream.as_raw_fd(), listener_read_event(key))?;
                            request_contexts.insert(key, RequestContext::new(stream));
                        }
                        Err(e) => eprintln!("couldn't accept: {}", e),
                    };
                    modify_interest(epoll_fd, listener_fd, listener_read_event(100))?;
                }
                ...
}

fn modify_interest(epoll_fd: RawFd, fd: RawFd, mut event: libc::epoll_event) -> io::Result<()> {
    syscall!(epoll_ctl(epoll_fd, libc::EPOLL_CTL_MOD, fd, &mut event))?;
    Ok(())
}
```

如果我们遇到了 100，我们就知道这是我们的 server，因为这是我们之前为它设置的硬编码的 key。因为我们只注册了对读就绪事件的兴趣，这意味着我们现在能 `accept` 一个传入的连接，而且不会阻塞。

如果我们没有得到通知就调用 `listener.accept()`，那这个调用将会阻塞直到连接到来。在此例中，我们被通知这里已经存在一个连接，我们对 `accept()` 的调用将会直接返回。

一旦我们接受了该连接，我们把连接到客户端的 `TCPStream` 也设置成非阻塞的。我们将 `key` 递增，以获得另一个独一无二的值并注册对该 `TCPStream` 的读就绪事件的兴趣。

因为我们是一个 HTTP 服务器，接受连接的下一步就是读取传入的请求。

最后，我们添加了一个条目到 `request_contexts` 哈希表里。这是一个非常原始的存储方式，用于保存所有活动连接的状态，稍后我们读取请求并写入响应时会用到它。

`RequestContext` 结构体有传入的连接的 `TCPStream` 和稍后我们读取请求要处理的两个字段。我们用 `key` 作为唯一键，将该 `RequestContext` 结构体放入哈希表中。

如前所述，因为我们使用 oneshot 事件（`EPOLLONESHOT`），我们需要重新注册我们对读就绪事件的兴趣，以得到后续的连接的通知。为此我们使用 `modify_interest` ，和 `add_interest` 做的一样，只是它使用 `EPOLL_CTL_MOD` 标志，因为该文件描述符之前就已被添加到了 epoll 实例。

好的，这就是接受连接的部分。下一步是将传入的 HTTP 请求读入一个缓冲区，然后写回一个请求 —— 全部以非阻塞方式。

```rust
fn main() -> io::Result<()> {
    ...
                key => {
                    let mut to_delete = None;
                    if let Some(context) = request_contexts.get_mut(&key) {
                        let events: u32 = ev.events;
                        match events {
                            v if v as i32 & libc::EPOLLIN == libc::EPOLLIN => {
                                context.read_cb(key, epoll_fd)?;
                            }
                            v if v as i32 & libc::EPOLLOUT == libc::EPOLLOUT => {
                                context.write_cb(key, epoll_fd)?;
                                to_delete = Some(key);
                            }
                            v => println!("unexpected events: {}", v),
                        };
                    }
                    if let Some(key) = to_delete {
                        request_contexts.remove(&key);
                    }
                }
    ...
}
```

如果我们得到了一个事件，但 `key` 不是 100，我们就知道该事件一定是被一个传入的连接所触发，因为这是我们唯一注册过兴趣的文件描述符。

我们定义了一个 `to_delete` 变量，如果请求完成，就将该 `RequestContext` 从哈希表中移除。这部分我们稍后讲。

我们在 `HashMap<u64, RequestContext>` 哈希表中检查是否存在给定的 `key` 的映射，如果有，我们就匹配 `events`，这是个位图，标识了哪些事件被触发了。

我们先处理 `read` 的情况，也就是 `EPOLLIN`。在此例中，我们在这个 `RequestContext` 上调用 `read_cb` 方法：

```rust
impl RequestContext {
...
    fn read_cb(&mut self, key: u64, epoll_fd: RawFd) -> io::Result<()> {
        let mut buf = [0u8; 4096];
        match self.stream.read(&mut buf) {
            Ok(_) => {
                if let Ok(data) = std::str::from_utf8(&buf) {
                    self.parse_and_set_content_length(data);
                }
            }
            Err(e) if e.kind() == io::ErrorKind::WouldBlock => {}
            Err(e) => {
                return Err(e);
            }
        };
        self.buf.extend_from_slice(&buf);
        if self.buf.len() >= self.content_length {
            println!("got all data: {} bytes", self.buf.len());
            modify_interest(epoll_fd, self.stream.as_raw_fd(), listener_write_event(key))?;
        } else {
            modify_interest(epoll_fd, self.stream.as_raw_fd(), listener_read_event(key))?;
        }
        Ok(())
    }

    fn parse_and_set_content_length(&mut self, data: &str) {
        if data.contains("HTTP") {
            if let Some(content_length) = data
                .lines()
                .find(|l| l.to_lowercase().starts_with("content-length: "))
            {
                if let Some(len) = content_length
                    .to_lowercase()
                    .strip_prefix("content-length: ")
                {
                    self.content_length = len.parse::<usize>().expect("content-length is valid");
                    println!("set content length: {} bytes", self.content_length);
                }
            }
        }
    }
...
}
```

首先，我们定义了一个临时的缓冲区用于读入数据，并且在我们保存的 `TcpStream` 上调用 `read()`。这里不会阻塞，因为我们已经被通知该文件描述符实际上可读了。

然后我们尝试将数据转为 utf8，查找 `content-length` 首部，并解析它的值。

注意：这只是一些非常愚蠢和最小的伪 HTTP 解析，这样做这非常不安全，这里只是为了方便实现和测试。

我们只对可以读取的数据量感兴趣，这样我们就能知道什么时候可以停止读取。一旦我们得到了内容长度，我们就把它放在 `RequestContext` 里。

然后我们将数据从临时缓冲区移动到 `RequestContext` 内部的缓冲区里。

之后我们检查是否读完了我们预期读取的所有数据，如果读完了，就再次调用 `modify_interest`，注册对写就绪事件的兴趣。这是因为一旦我们完成了对传入的连接的读取，下一步就是处理有效载荷，并对客户端做出响应。

如果我们仍有数据要读，就重新注册对读就绪事件的兴趣。

在注册了对写就绪事件的兴趣后，我们期待 `EPOLLOUT` 标志会被设置：

```rust
fn main() -> io::Result<()> {
    ...
                            v if v as i32 & libc::EPOLLOUT == libc::EPOLLOUT => {
                                context.write_cb(key, epoll_fd)?;
                                to_delete = Some(key);
                            }
                            v => println!("unexpected events: {}", v),
    ...
}
```

如果 `EPOLLOUT` 标志被设置了，我们就调用 `write_cb`，并将该 `RequestContext` 标记为待删除，这是因为我们完成数据写入后，我们的工作就做完了，该进行清理工作了。

```rust
impl RequestContext {
...
    fn write_cb(&mut self, key: u64, epoll_fd: RawFd) -> io::Result<()> {
        match self.stream.write(HTTP_RESP) {
            Ok(_) => println!("answered from request {}", key),
            Err(e) => eprintln!("could not answer to request {}, {}", key, e),
        };
        self.stream.shutdown(std::net::Shutdown::Both)?;
        let fd = self.stream.as_raw_fd();
        remove_interest(epoll_fd, fd)?;
        close(fd);
        Ok(())
    }
...
}
...
fn close(fd: RawFd) {
    let _ = syscall!(close(fd));
}

fn remove_interest(epoll_fd: RawFd, fd: RawFd) -> io::Result<()> {
    syscall!(epoll_ctl(
        epoll_fd,
        libc::EPOLL_CTL_DEL,
        fd,
        std::ptr::null_mut()
    ))?;
    Ok(())
}
```

这个 `write_cb` 比 `read_cb` 更简单。因为我们收到了套接字可写的通知，所以我们可以调用 `stream.write()` 用硬编码的 HTTP 响应和客户端友好地打个招呼。

在此之后，我们需要清理该连接。首先，我们在 `TcpStream` 上调用 `shutdown` 以关闭连接。然后移除对文件描述符的兴趣，关闭文件描述符，到这里我们就做完了。

由于我们设置了 `to_delete`，这个 `RequestContext` 也会被移出哈希表。

让我们看看能否有效工作。

## 测试

我们想要确保服务器可以同时接受并处理多个请求，即使有很多个也不会变慢。

有一件事要注意，因为我们完全没有做错误处理，所以如果发生任何 IO 错误发生（例如，在我们读取的时候取消传入的请求），程序将会 panic。

因此，让我们向服务器发送一些运行的更长的请求：

```bash
while true; do curl --location --request \
POST 'http://localhost:8000/upload' \
--form 'file=@/home/somewhere/some_image.png' \
-w ' Total: %{time_total}' && echo '\n'; done;
```

这个命令在一个无限循环中不停地发送一个文件（最好是比较大的，例如几个 MB）。我们可以在多个终端中运行该脚本，这样我们就有了一个高负载的 IO。

此外，通过 `-w %{time_total}` 选项，我们给这个请求计时，每个成功的请求会打印这样的内容：

```
Hello Total: 1,010951
```

“Hello” 是响应，然后是请求所花费的总时间，在该例中大约是 1 秒钟。

有一点需要注意的是，我们的服务器确实在正常地服务请求并且不会变慢。此外，如果我们检查服务器日志，我们会看到一些像这样的：

```
...
requests in flight: 6
...
```

这告诉我们，我们确实用单个线程同时处理了几个请求。此外，如果你检查你的 `while curl` 脚本，你会发现请求仍然花费着相同的时间。

这证明了我们的解决方案有效，并且我们确实用非阻塞 IO 非常快地并发处理了许多高负载 IO 请求。棒极了！

完整示例代码在[这里](https://github.com/zupzup/rust-epoll-example)。

## 总结

我在一个较高的层级上对非阻塞 IO（或者说异步编程）的使用已经有一段时间了，并且是以各种形式的，比如 futures、promises 和基于回调的 APIs。

然而，我从来没有理解底层到底发生了什么，以及这个模型为什么有效。我仍不觉得自己了解所有的细节，但是研究并编写这个例子让我有了更好的理解。

我必须对那些写出易学的教程和代码的人说声感谢，这些教程和代码可以在下文的参考资料一节被找到。

我希望我能够为你提供一些新的理解，给你启迪，让你感到有趣。:)

我的计划是继续深入探索 Rust 异步编程。

## 参考资料

- [Code Example](https://github.com/zupzup/rust-epoll-example)
- [Asynchronous Programming Under Linux](https://unixism.net/loti/async_intro.html)
- [Linux Applications Performance: Part VII: epoll Servers](https://unixism.net/2019/04/linux-applications-performance-part-vii-epoll-servers/)
- [Exploring Async Basics](https://cfsamson.github.io/book-exploring-async-basics/introduction.html)
- [Epoll, Kqueue and IOCP](https://cfsamson.github.io/book-exploring-async-basics/6_epoll_kqueue_iocp.html)
- [Polling Rust Library](https://github.com/stjepang/polling)
- [Mio](https://github.com/tokio-rs/mio)

[epoll_create]: https://man7.org/linux/man-pages/man2/epoll_create1.2.html
[epoll_wait]: https://man7.org/linux/man-pages/man2/epoll_wait.2.html
[epoll_ctl]: https://man7.org/linux/man-pages/man2/epoll_ctl.2.html
