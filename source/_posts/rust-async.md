---
title: Rust 异步编程（四）—— 基本的 future， stream 和 sink 应用
date: 2023-05-31 23:20:28
tags:
categories:
- Rust
---

futures 库是 Rust 最最基本的异步编程库，这篇文章将总结大部分 futures 库的使用方法，其底层原理将一带而过，见前面的文章。我们将从最简单的 Future 开始，最后到最复杂的应用。其过程是我们在实际项目中最常用的。本节需对前面的异步编程底层模型有所了解，且对 Rust 基本的用法有所了解（刷 leetcode 容易题没有觉得特别难就能用 Rust 的方法写出来的水平即可。）

网上很多这方面的文章，但要么复杂庞大，导致难以理解（比如官方文档，或者一些一股脑堆代码的长篇博文），要么过于简单，讲不出重点。这里力求用最最最简单的代码去把最最最核心的异步编程基础应用讲出来，相信我，看完这篇文章，对 Rust 的异步编程绝对有核心的理解。

<!--more-->

## Future

获得一个 Future 对象有两种方法，最简单的就是定义一个 async 开头的函数，其函数体就是 Future 执行的内容。但遇到复杂的流程，还是需要自己去 impl Future trait 的，下面我们实现一个 Future trait，其功能很简单，就是过一段时间返回一个 String 对象：

```rust
struct MyFuture {
    duration: u64,
    start_time: u64,
    name: String,
}

impl MyFuture {
    pub fn new(duration: u64, name: String) -> Self {
        MyFuture {
            duration,
            start_time: SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs(),
            name,
        }
    }
}

impl Future for MyFuture {
    type Output = String;

    fn poll(self: std::pin::Pin<&mut Self>, cx: &mut std::task::Context<'_>) -> std::task::Poll<Self::Output> {
        let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
        let pass = now - self.start_time;
        if pass < self.duration {
            cx.waker().wake_by_ref();
            return std::task::Poll::Pending;
        }

        println!("{} is completed, {now}", self.name);
        return std::task::Poll::Ready(Ok(self.name.clone())); 
    }
}
```

其 poll 函数，获取当前时间，然后和初始化时间相比，若超过特定时间，则返回 name，否则调用 `cx.waker().wake_by_ref()` 将这个 Future 对象放回就绪队列待下一次 executor 取出来重新执行 poll 函数，直到返回 Ready 为止。也就是说，如果 `pass < self.duration` 不满足，这这个 Future 对象会一直被放回就绪队列， poll 函数会不停的被调用，直到 ready 返回。

此时，调用方会一直阻塞在 await 上，直到 ready可以返回：

 ```rust
 fn main() {
     futures::executor::block_on(async {
         let jack = MyFuture::new(3, "jack".to_string());
         let rose = MyFuture::new(5, "rose".to_string());
         let janey = MyFuture::new(2, "janey".to_string());
 
         jack.await;
         rose.await;
         janey.await;
     });
 }
 ```

上面的代码，会顺序执行 jack， rose 和 janey，也就是一共需要 3 + 5 + 2 = 10 秒钟执行完，且是顺序执行。因为 await 会触发对应的 poll 调用，而 poll 中不被重新返回就绪队列的路径是 `pass < self.duration ` 不成立，返回 ready。

那么这就是基本的异步编程。

于是很多人就会有疑惑：这不是异步编程， 因为 jack，rose 和 janey 是顺序执行的，这只是换个法子在同步执行。这么说是完全正确的，的确这是同步的，jack 没有完成是轮不到 rose 的。这里不去过多的解释，我们先把 Future 的能力记下，继续往下。



## Stream

stream 是什么？我们最熟悉的 iterator 就是 stream：

```rust
        let result = vec![1, 2, 3].into_iter().map(|item| item << 1).collect::<Vec<_>>();
        println!("{result:?}");
```

输出：

```bash
[2, 4, 6]
```

那么我们用 stream 实现一次：

```rust
        let result = futures::stream::iter([1, 2, 3]).map(|item| item << 1).collect::<Vec<_>>().await;
        println!("{result:?}");
```

输出是和之前一样的，总之，stream 就是 iterator，和之前唯一不同的是，我们的 collect 调用有一个 await 尾巴。这是因为 Stream 是异步编程，它是基于 Future 实现的，所以只有调用了 await 才干活，否则，只是返回一个 Future 对象。

所以，stream 就是 iterator，我们把 future 对象放到 stream 就可以使用 stream 的方法调用里面的 future await 了：

```rust
fn main() {
    futures::executor::block_on(async {
        let jack = MyFuture::new(3, "jack".to_string());
        let rose = MyFuture::new(5, "rose".to_string());
        let janey = MyFuture::new(2, "janey".to_string());

        let result = futures::stream::iter([jack, rose, janey]).collect::<Vec<_>>().await;
        for item in result {
            item.await;
        }
    });
}
```

我们把 stream 转成一个集合，然后遍历这个集合，因为里面是 future，所以可以调用 await。

但这么写好像没获得什么好处。其实还是有好处的，就是上面的 3 到 5 行和 7 到 10 行解藕了，也就是写 stream 的人并不需要知道 jack，rose 和 janey。这就是 stream 带来的第一个好处：只需要知道里面的东西是一个 future 和其返回值就行，至于具体是什么 future（是 jack，rose还是 bob， bill 统统无需关心。），怎么实现的，不需要关心。

这个好处似乎和异步编程没什么关系，我们继续往下。

如果 stream 是专门为 future 设计的 iterator，那应该有写特殊方法帮我们出发 future，比如 buffered：

```rust
fn main() {
    futures::executor::block_on(async {
        let jack = MyFuture::new(3, "jack".to_string());
        let rose = MyFuture::new(5, "rose".to_string());
        let janey = MyFuture::new(2, "janey".to_string());

        let buffer = futures::stream::iter([jack, rose, janey]).buffered(1024);
        let result = buffer.collect::<Vec<_>>().await;
        println!("{result:?}");
    });
}
```

和之前直接用 collect 把 future 打包进 Vec 相比，这里先用 buffered 函数转成 buffer 对象，然后再去 collect，这时的 result 就是 future ready 返回的结果了，也就是说我们只需要调用一次 await，就可以把 stream 的 future 都触发，并且把结果放进一个 collect 里面。

如果运行上面的代码，会发现一个现象，stream 的 future 同时运行了，不再是先运行前一个再触发后一个了。我们终于得到了一个并发编程的代码，关键是，我们没有使用线程。这得益于之前讲的异步编程模型，简单说，就是三个 future 都被放到就绪队列，executor 去不停的检查它们的返回值，若 pending，由于前面说的 wake 调用会被重新放回就绪队列，若 ready 则返回结果放入 Vec。这三个 future 不停的被遍历，这也就实现了并发。

尽管现在已经是并行执行三个 future，但我们可以不可以做到，先完成的 future 先输出呢？当然可以：

```rust
fn main() {
    futures::executor::block_on(async {
        let jack = MyFuture::new(3, "jack".to_string());
        let rose = MyFuture::new(5, "rose".to_string());
        let janey = MyFuture::new(2, "janey".to_string());

        let buffer = futures::stream::iter([jack, rose, janey]).buffer_unordered(1024);
        let result = buffer.collect::<Vec<_>>().await;
        println!("{result:?}");
    });
}
```

相比之前，我们用了 `buffer_unordered` 这个方法，也就是无序的收集结果，因此，像 janey 这样只需要 2 秒钟就完成的任务就先输出了。

上面的 1024 表示 buffer 可以并发执行多少 future，若改成 1，那么就相当于顺序执行了，而改成 0，相当于就绪队列放不下任何 future，这样将永远阻塞。实际上， buffer 是一个特殊的 stream，stream 能做的它也能做，不同点是，buffer 直接给出的是 future 的运行结果。



## Sink

前面我们用了 stream 去并发执行 future，但有个问题需要解决，即结果都是所有的 future 都运行完后才返回的（尽管它们是并发执行），是否可以做到来一个结果就输出一次呢？当然可以，我们需要 sink 这个概念帮我们完成。

sink 不是什么特殊的东西，它其实是我们入门 Rust 时学习的、最熟悉不过的 `mpsc::channel ` 概念相关，即 sender 和receiver。而 sink 其实就是 sender。我们下面就把 sink 换成 sender 来说，会发现其实 sink 很好理解。

既然是 sender，那么就知道 sender 一边发结果，receiver 一边收结果，只要有结果，就及时 send 给 receiver：

```rust
#[tokio::main]
async fn main() {
    let jack = MyFuture::new(3, "jack".to_string());
    let rose = MyFuture::new(5, "rose".to_string());
    let janey = MyFuture::new(2, "janey".to_string());

    let mut buffer = futures::stream::iter([jack, rose, janey]).buffer_unordered(1024);

    let (mut sender, mut receiver) = futures_channel::mpsc::channel(1024);

    let handle = tokio::task::spawn(async move {
        loop {
            let result = receiver.next().await;
            println!("{result:?}");
        }
    }); 

    sender.send_all(&mut buffer).await;

    handle.await;
}
```

和之前相比，我们加入了 `futures_channel::mpsc::channel(1024)` 来生成 sender 和 receiver，同时开启一个线程用于接受结果，而主线程，调用 `send_all`  方法将我们的 future 执行并把结果发送出去，哪个 future 有了结果，哪个就被发送。

如果运行，会发现 future 的结果都能及时打印出来了。

这里有几个要说的：

1、这里开启了线程，当然可以把 sender 和 receiver 统一封装到一个 future 类里面去在同一个线程中并发执行，但并发如果头铁就是不想多线程，那么是很僵化的，多线程可以利用多 CPU 的能力，future await 的引入，主要是1、解决 IO 阻塞导致上下文切换的问题；2、降低编写并发代码的复杂度。

2、上面的 `mpsc::channel(1024)` 表示发送队列的大小。这里要注意以下三者的关系：buffer 的大小，channel 的大小和 receiver 的处理能力。假设有这么个极端情况，receiver 忙死了，那么 channel 通道就会被塞满，进而导致 buffer 的并发量下降到 0，虽然有 `unbounded` 替代 channel，但个人认为这比较危险，因为如果整个系统堵死了，可以被监控到，但使用 `unbounded` 是要等到内存耗尽才可能触发到系统异常，这个时候再去处理已经晚了。

3、注意一个小细节，即 future 的返回值从 String 变成了 `Result<String, SendError>` ，这里没贴出这个变化的代码。因为我们要时刻监控 channel 队列，如果 send 失败，应该有所处理，因此 sender 要求 future 必须用 Result 来做 future 的返回值。

4、不要自己实现 sink trait。尽管可以这么做，但不要这么做，因为大概率，我们写的代码都不如官方的实现优秀。



## 总结

这篇文章引入了 stream（一个异步 iterator），buffer（一个特殊的 stream，collect 返回的是 future 的结果） 和 sink（sender，让我们使用 receiver 及时处理 future 的输出结果） 的概念，是 Rust 异步编程基本概念，后面会分享更多细节。
