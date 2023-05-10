# starcoin 异步模型分析

## 基于 actix

### 官方的 actix 

首先应该先根据官方的文档，熟悉基本的 actix 框架，包括：actor，address，contex，arbiter 和 sync arbiter 这些基本概念。

官方文档和示例代码见：https://actix.rs/docs/actix

### starcoin 中基于 actix 的异步消息处理框架

#### 综述

starcoin 的异步消息框架是对 actix 进行了一层封装，主要思想是：一个服务一个线程。

#### ServiceActor\<S\>

其 impl 了官方的 Actor 和 Handle trait，也就是所有的异步消息都会走 ServiceActor\<S\>，其中 S 是我们应用层真正实现业务的类，例如，如果我们想新建一个 XXX service，那么只需要：

```rust
// 定义service 和 创建工厂类
// impl ActorService trait，这样才可以给到 ServiceActor 中的 proxy 控制
// impl EventHandler<S, E> 和 ServiceHandler<S, R> 处理消息
struct XXXService; 

// 实现 ServiceFactory<S>::create 函数，生成 serivce
struct XXXServiceFactory; 

// 使用，在 init_system() 函数中，通过 registry 对象进行注册启动 service
registry.register_by_factory::<XXXService, XXXServiceFactory>().await?; // 此处会开启一个线程处理XXX Service 感兴趣的消息
```

因此，ServiceActor\<S\> 主要还是用于完成对 Actor 的封装，黏合 Actor 和starocin service。

ServiceActor\<S\> 内部主要封装了一个 proxy （ServiceHandlerProxy\<S\>），它封就是我们的服务 proxy，任何 request 都是通过这个 proxy 类再分发到对应的 service 上去的。这个属于 starcoin 基于 actix 的异步消息的内部实现，一般是感知不到。

#### EventHandler\<S, M\> 和 ServiceHandler\<S, R\>

starcoin  的消息分为 event 和 service 两种，本质上没啥不同，都是 actix 消息，前者应该是用于事件监听（例如开始同步，结束同步），没有返回值，后者是用于消息发送，有返回值。我们定义 XXXService 后可以 impl 这两个 trait，其中 S 是我们的 service， M 和 R 是我们自定义的 message struct。实现 handle 函数处理发来的消息。

#### ServiceContext\<'a, S\>

我们还会遇到 service conntext 类，这个类其实就是封装了 actix 的context，即可以通过这个 context 获得 service 的地址，启动，重启 service 本身。任何 service 都可以在 handle 消息的时候拿到这个 context。

比 actix 更多的是，它还封装了一个 service cache，里面放了两样东西：

1、一个特殊的 service： register serice；

2、其它对象的引用（放在 box 中），包括：配置，其它 service的等，用 any 这个类型放的；

#### ServiceRef\<S\>

其实就是封装了 actix 的 address，用于给服务发消息，还可以启动停止通知等操作，因为 service 之间经常交互发消息，因此会比较常用。

### 一个特殊的service：RegistryService

RegistryService 比较特殊，它是第一个创建的 service，它的任务就是创建一个线程（用 actix 的 arbiter）其它 service，并把其它 service 放到它的 context 中缓存起来，而且它也存放了一些全局属性，比如 node config，vm config，甚至 genesis 等对象。

主要有几个关键的功能：

| 函数名（同步）/消息（异步）                        | 作用与实现                                                   |
| -------------------------------------------------- | ------------------------------------------------------------ |
| register/RegisterRequest                           | 创建一个线程，拉起一个 service                               |
| register_by_factory（这是一个asyn 函数仅支持异步） | 同 register，但需要设计 service 方提供一个 factory类（实现ServiceFactory\<S\> 这个 trait，S 是 service 类） |
| get_shared/GetShardRequest                         | 获取任何对象，一般是全局唯一的对象，比如配置，genesis，serivice 对象。里面用 any 存放 |
| put_shared/PutShardRequest                         | 存放任何对象，一般是全局唯一的对象，比如配置，genesis，serivice 对象。里面用 any 存放 |
| service_ref/ServiceRefRequest                      | 获得某个 service 的 ServiceRef，即 address                   |

这个 register serivice 的 address 会被放在 context 中，即每个 service 都可以引用它，给它发消息进行交互（异步使用），也可以同步调用它直接的实现函数（同步使用）。因此，RegistryService 相当于一个全局 service 容器，通过它（异步给它发消息 ServiceRefRequest\<S\>，同步直接调用 get_shared），可以获得其它 service 的 ServiceRef\<S\> 这样就可以和其它服务交互了。其关键代码见：commons/service-register/src/service_registry.rs。

对 RegistryService 对象的初始化构建流程目前都集中在启动过节点时的  init_system 函数 （node/src/node.rs）中。

### 调试方法与经验

如果是同步调用，比较简单，如平常一样流水线般跟踪即可。

如果是异步调用：可以跟踪 request 的处理，看哪里处理了 request（即看 impl 了event handle或者 service handle 的地方）。



## 基于 futures

### futures 的官方文档

基于 futures 的比较复杂，官方文档和例子比较少，也难懂，所以建议入门可以参考 tokio 的官方文档熟悉一下（相对 futures 简单）：

https://tokio.rs/tokio/tutorial

入门后再看 futures 的官方文档：

https://docs.rs/futures/latest/futures/

可以仔细阅读注释（文档）来学习 futures。

### 理解 async 和 await

tokio 的官方文档有很详细很生动的解释 async 和 await，建议跟着敲一遍里面的代码体会一下 async 和 await，这里简单总结一下：

async：即生成一个 future，如果函数被定义为 async 函数，则说明函数返回值是一个 future；一个 lambda 被定义为 async，也变成 future。

await：一个 future 对象的特殊调用，即尝试执行一个 future，若 future 因为被挂起，则等待唤醒，唤醒后，继续往下执行。重点理解这篇文章：https://tokio.rs/tokio/tutorial/async。简答的说，即：调用 await 会触发 future 类的 poll 调用，若 poll 函数返回 Ready 状态，那么 await 就返回，且带上 Ready 关联的返回值。若 poll 函数返回 Pending 状态，那么 await 继续挂起，即阻塞在 await 中，直到 有人调用 poll 中的 conntext waker 的 wake 函数，此时 poll 函数会重新被拉起，看 await 是否又可以继续执行：



了解了基本的 future await 概念后就可以去看 futures的 stream 了。

### stream 异步编程

#### Stream

futures 提供了一种特殊的异步模型，即 stream，它是基于 future 异步（注意 futures 和 future，前者是一个crate，后者是一个类）类实现的。

stream 即流，类似比较熟悉的概念即 iterator，即 iterator 有的函数 stream 也有（比如，map，collect 等），也就是说 stream 一种集合，如果我们看 stream 的定义，会发现有一个和 next（iterator 的函数） 和 poll（future 的函数） 类似的函数：

```rust
 fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
```

如果我们调用了stream 的 await，那么它的 poll_next 就会被触发，其返回也是 Ready 或者 Penndig，关联集合中的某一项（Item）。当所有的集合元素都被 poll_next 一遍后，就返回 Ready(None)。相比之前 for 循环同步遍历 iterator，stream 这些动作都是异步执行，如下（伪代码）：

```rust
let s = stream::new();
while let some(item) = s.await { // await 触发 next_poll 调用，有 item 则返回，没 item 则挂起等待
  println!(item);
}
```

特别的，若 stream 中的 item 都是 future 的话，那么我们就可以实现一种异步批量执行 future 的异步模型：

如果有一个任务 task，它包含了若干个子任务 sub tabsk，且需要异步执行，那么，此时可以用 stream 来控制，例如第一次调用 poll_next 生成第一个子任务，子任务结束的时候 wake  起 stream 继续 poll next，直到执行结束。这里的子任务即 future。

简单说，就是相比单个 future， stream 实现了批量异步执行子任务。如下（伪代码）：

```rust
let s = stream::new();
while let some(future) = s.await { // await 触发 next_poll 调用，有 future 则返回，没 future 则挂起等待
  let result = future.await;	//  触发 poll，若返回 pending，则挂起
  println!(result);
}
```



（待补充图）

#### iter 函数 和 buffered 函数

实际上，stream 并不是异步类，即没有 stream.await 这样的调用。想获得异步对象，需要调用 stream 的 iter 或者 buffered 函数。

1、stream 的 item 不是 future 对象，使用 iter，iter 函数返回 Collect 对象；它帮我们实现了异步 await item 的流程，关键源代码如下：

```rust
impl<St, C> Future for Collect<St, C>
where
    St: Stream,
    C: Default + Extend<St::Item>,
{
    type Output = C;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<C> {
        let mut this = self.as_mut().project();
        loop {
            match ready!(this.stream.as_mut().poll_next(cx)) {  // 调用了 stream 的 poll_next，异步拉取 item
                Some(e) => this.collection.extend(Some(e)),
                None => return Poll::Ready(self.finish()),
            }
        }
    }
}
```

2、stream 的 item 是future 对象，使用 buffered，buffered 函数返回 Buffered 对象；它帮我们实现了启动 future await 的流程：

```rust
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let mut this = self.project();

        // First up, try to spawn off as many futures as possible by filling up
        // our queue of futures.
        while this.in_progress_queue.len() < *this.max {
            match this.stream.as_mut().poll_next(cx) {
                // 已经 ready 有对象实体的，进入执行队列
                Poll::Ready(Some(fut)) => this.in_progress_queue.push_back(fut), 
                Poll::Ready(None) | Poll::Pending => break,
            }
        }

        // 拉起执行队列中的 future 
        // Attempt to pull the next value from the in_progress_queue
        let res = this.in_progress_queue.poll_next_unpin(cx);
        if let Some(val) = ready!(res) {
            return Poll::Ready(Some(val));
        }

        // If more values are still coming from the stream, we're not done yet
        if this.stream.is_done() {
            Poll::Ready(None)
        } else {
            Poll::Pending
        }
    }
```

可以看到，poll_next 中会 while 遍历 future，然后若这些 future 上一次都是返回 ready，则放入队列中，然后依次触发队列中的 future（poll_next_unpin会调用到 future 的 poll）。

以上可以参看我的例子中的 test_stream_vec 和 test_stream 两个测试函数：

https://github.com/jackzhhuang/myactix/blob/master/src/main.rs

当然，后面方便讨论还是会说 stream.await，知道需要使用 iter 或者 buffered 来启动异步遍历流程就好。

#### Sink

stream 的 iter 和 buffer 函数返回 future 异步的去遍历 item，但其返回是所有的 item 集合，是一次性的，也就是 stream 没有给我们展现中间过程的异步处理。如果我们想每处理一个 item 就做一些事情，就需要 sink trait。sink 可以让我们监控整个 stream 内部遍历 item 的动作，翻看 sink 的 trait 声明会看到：

```rust
pub trait Sink<Item> {
    /// The type of value produced by the sink when an error occurs.
    type Error;

    // start_send 之前调用
    fn poll_ready(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
		
    // 处理某个 item 之前调用
    fn start_send(self: Pin<&mut Self>, item: Item) -> Result<(), Self::Error>;

  	// 所有 item 都过一遍后调用，若还有 item 处于挂起，则返回 pending，否则若执行返回 ready
    fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;

  	// 结束前调用
    fn poll_close(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
}
```

有了 sink，就可以监控对 item 的处理，例如（伪代码）：

```rust
let s = stream::new();

// stream 转为 map<stream, future>，这里的 future 即在 sink 中异步去调用 stream await
let buffered = s.buffered().map(ok).collect(); 
let sink = sink.send_all(buffered);
sink.await; // 异步调用 stream.await，每处理一个 item，就回调一下 sink trait 对应的函数
```



### starcoin 的 stream-task 异步框架

了解完官方的 futures 后可以开始看 starcoin 封装的 stream task 框架了。

#### 综述

实现了一个异步任务框架，只需要实现 TaskState 和 Generate 这两个 trait 就可以异步执行并在错误的情况下可以重试。

#### FutureTaskStream\<S\>

即 stream 类，我们在里面封装了 TaskState，stream.await 调用的时候，就会在 poll_next 里面调用 TaskState 的 new_sub_task 并返回 future 对象，最终异步执行 TaskState 的 future。每次执行完，都会调用 TaskState 的 next 函数，next 函数会根据当前 TaskState 的执行状态生成新的 TaskState 供 stream 执行。

#### trait TaskState

即 FutureTaskStream\<S\> 里面的 S。封装 future，在 new_sub_task 中返回当前的 future，next 则负责返回下一个 future，在 stream 中依次执行。

实际上，整个异步框架了解以上两个类就足够，以下两个主要用于辅助以上两个类运作。

#### trait Generator 和 struct TaskGenerator\<S, C\>

generate 函数主要拉起了整个异步流程，即初始化 TaskState，stream 和 sink。其把这些流程也打包到一个 future 中，然后调用放会 await 拉起异步 stream 流程。

#### TaskFuture\<Output\> 

由 Generator 返回，对 future 进行了封装，可以获得 future 的 handle，与 TaskState 不同的是，TaskState 是用于批量执行子任务的，而 TaskFuture\<Output\>  更像是一个单独的 future 封装。

#### UML







