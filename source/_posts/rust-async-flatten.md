---
title: Rust异步编程(五)—— buffer 和 flatten
date: 2023-06-10 15:47:30
tags:
categories:
- Rust
---

日积跬步，慢慢积累异步编程的知识。

未来异步编程将朝两个方向：

1、异步编程的应用，主要对 futures 等 Rust 库进行探索，比如今天要讲的 buffer 函数和 flatten 函数；

2、对底层异步实现进行探索，比如实现自己的Arc，Mutex。

<!--more-->

## buffer 函数

futures 库中的 buffer 函数上一次讲过，它其实是一个特殊的 stream，一般 stream 调用 next（这里顺带一提，next 会取出元素），返回的是其中的元素，而 buffer 调用 next，则是返回元素调用 await 后的返回值，因此，stream 想变成 buffer 这个特殊的 stream，必须保证其中元素是一种 future。

关于 buffer 函数，有三个需要注意的：

### buffered(self, n: usize) 函数

它是一个按顺序执行 future 的函数，即前一个 future 没有结果前，后面的 future 都处于等待状态，从这点上讲，它没有并发效果。但能保证任务顺序执行，对于有任务顺序执行的异步需求是一个不错的函数。

其中的 n 表示 buffer 的大小，个人目前觉得这个大小对于顺序执行来说没有太大的影响。

### buffer_unordered(self, n: usize) 函数

和 buffered 相比，`buffer_unordered(self, n: usize)` 则是并发执行，即我们的 executor 会遍历 buffer 中的所有 future，并询问其执行状态。因此，n 将表示其并发执行的数目。因此，如果 n = 1，此时就相当于顺序执行了。

### buffer 的大小与 channel 的大小

上一节说了 future-stream-sink 模型（https://www.jackhuang.cc/2023/05/31/rust-async/），其中有两个地方需要设置 buffer 大小，一个是 stream 生成 buffer 的时候的大小，一个是 sink 在异步接收结果的时候 buffer 的大小。

这两个大小有什么联系呢？前者表示并发数量，后者表示结果缓存的数量，正常情况下，没有绝对的联系，但试想一个极端的情况，如果并发数量远大于 channel 的通道 buffer，那么 channel 的 buffer 会被瞬间塞满，出现处理结果遭遇瓶颈。由于 buffer 的结果不能发送给 channel，stream 这边的 buffer 也遭遇瓶颈，最终阻塞整个异步流程，继续以上一节为例：

```rust
#[tokio::main]
async fn main() {
    let jack = MyFuture::new(3, "jack".to_string());
    let rose = MyFuture::new(5, "rose".to_string());
    let janey = MyFuture::new(2, "janey".to_string());
    let bob = MyFuture::new(3, "bob".to_string());
    let ryn = MyFuture::new(3, "ryn".to_string());

    let mut buffer = futures::stream::iter([jack, rose, janey, bob, ryn]).buffer_unordered(2);

    let (mut sender, mut receiver) = futures_channel::mpsc::channel(1);

    
    let handle = tokio::task::spawn(async move {
        loop {
            let result = receiver.next().await;
            println!("{result:?}");
            loop {
                
            }
            match result {
                Some(_) => continue,
                None => break,
            }
        }
    }); 

    let send_result = sender.send_all(&mut buffer).await;
    println!("sending is done!");

    handle.await;
}

```

 上面的代码中，`buffer_unordered`  的大小为2，而 `channel` 大小为 1，此时有5个 future 等待执行，在处理结果的 loop 中，我加了一个死循环，使其只能处理一个结果，造成两个 buffer 堆积，首先 channel 的 buffer 结果得不到处理，造成 send 阻塞，接着 stream 这边的 buffer 产生结果后无法送给 channel，也阻塞，最终整个系统都停止运转了。可见，如果两边的处理效率不同，较慢的那一个会成为整个系统的瓶颈。

## flatten 函数

buffer 函数会触发 stream 的 future，返回 future 的结果。而 flatten 函数结合 buffer 的使用，还可以处理 future 返回 future 的情景。即，如果一个 future 返回了另一个 future，我们可以使用 flatten 函数获得最后的 结果，而忽略中间过程。例如我们多增加一层 future，套用在原来的 future 上：

```rust
#[derive(Clone)]
struct MyFuture2 {
    duration: u64,
    start_time: u64,
    name: String,
}

impl MyFuture2 {
    pub fn new(duration: u64, name: String) -> Self {
        MyFuture2 {
            duration,
            start_time: SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs(),
            name,
        }
    }
}
impl Future for MyFuture2 {
    type Output = Pin::<Box<MyFuture>>;

    fn poll(self: std::pin::Pin<&mut Self>, cx: &mut std::task::Context<'_>) -> std::task::Poll<Self::Output> {
        let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
        let pass = now - self.start_time;
        if pass < self.duration {
            cx.waker().wake_by_ref();
            return std::task::Poll::Pending;
        }

        return std::task::Poll::Ready(Box::pin(MyFuture::new(self.duration, self.name.clone()))); 
    }
}
```

上面的代码中，MyFuture2 返回了之前的 Future，此外，我们的 MyFuture 的结果需要修改成标准的 Result<T, SendError>，这样我们的 flatten 函数才能处理最后一个 future 结果：

```rust
impl Future for MyFuture {
    type Output = anyhow::Result<String, SendError>;

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

此时，我们的 main 函数变成了这样子：

```rust
#[tokio::main]
async fn main() {
    let jack = MyFuture2::new(3, "jack".to_string());
    let rose = MyFuture2::new(5, "rose".to_string());
    let janey = MyFuture2::new(2, "janey".to_string());

    let mut buffer = futures::stream::iter([jack, rose, janey]).map(|fut| fut.flatten()).buffer_unordered(3);

    let (mut sender, mut receiver) = futures_channel::mpsc::channel(2);

    
    let handle = tokio::task::spawn(async move {
        loop {
            let result = receiver.next().await;
            println!("{result:?}");
            match result {
                Some(_) => continue,
                None => break,
            }
        }
    }); 

    let _ = sender.send_all(&mut buffer).await;
    println!("sending is done!");

    handle.await;
}
```

可以看到，和之前相比，我们在从 stream 转为 buffer 的时候，多了一步 map，即 stream 的 MyFuture2 对象调用 一次 flatten，这样 buffer 就会返回最终的 future 结果，不需要我们外部再处理一层 future 逻辑。
