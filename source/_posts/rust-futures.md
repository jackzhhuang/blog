---
title: Rust 异步编程（二）：Future 和 Wake 
date: 2023-02-20 09:56:48
tags:
categories:
- Rust
- Concurrency
---

上一节我们大概知道了一些异步编程概念，我们快速简单的复习一下：

1、Rust 依靠生产者和消费者模型来实现异步编程；

2、被定义为asyn 的函数，其返回值为一个 future，调用它的 await 方法会触发函数的执行；

3、 因此，IO 线程完成工作后给主线成发送消息，触发 await，从而实现异步。

以上是上一节讲的最最简单的异步编程模型，但 Rust 异步编程并不是这么简单，因为上一节的模型中，我们没有模拟 IO 阻塞，因此，await 实际上是不会阻塞的，但 Rust 异步编程中，恰恰是因为 await 中有 IO 操作会产生阻塞而设计的。

那么，await 到底是怎么用的呢？更关键的是，如果 await 是阻塞的，那么是怎么做到一个线程的情况下切换上下文的呢？如何做 await 的 IO 操作完成后继续往下执行的呢？

本节将会把 await 展开，引入关键的两个 trait： Future 和 Wake，把 Rust 的异步编程模型说清楚。

<!--more-->

## 异步函数改写成 Future

上一节中，我们在函数前加了一个 async 关键字，此时会发现函数会变成一个feture，即我们的代码如下：

```rust
async fn SayHelloInPending -> String {
	// ...
} 
```

实际上，编译器会把它变成：

```rust
impl Future for SayHelloInPending {
    type Output = String;

    fn poll(self: std::pin::Pin<&mut Self>, cx: &mut std::task::Context<'_>) 
  				-> std::task::Poll<Self::Output {
      // ...
  }
 
```

也就是，任何一个异步函数，都会被化成一个个 future。future trait 的定义如下：

```rust
pub trait Future {
  type Output;
  fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

其关键的函数就是poll，poll 主要是返回一个Poll enum，它要么是 Ready(T)，表示阻塞完成，可以继续执行，T 即函数的返回值，或者Pending 表示函数依然在等待 IO 完成。

因此，我们只需要把定义的函数写成一个 struct，struct 实现 Future trait，并实现函数需要的阻塞 IO 逻辑即可，当然这个逻辑是在线程中执行，因为不能阻塞 main 线程，main 线程调用 Future trait 的 poll 的会检查阻塞的状态，若未完成，这个 poll 返回 Pending，否则返回 Ready(T)。

例如我们要实现一个 n 毫秒后返回 completed 字符串的函数，main 主线成会不等这个 n 毫秒，而是继续处理其它已经就绪的 Future，当 n 毫秒完过后，会把 Future 放入队列等待主线程 调度。

我们的异步函数会被写成这样（此时不再使用 async）：

```rust
struct SayHelloInPending {
    shared_data: Arc<Mutex<SharedData>>,
}

struct SharedData {
    completed: bool,
    waker: Option<Waker>,
}

impl SayHelloInPending {
   fn new(millis: u64) -> Self  {
        let task = SayHelloInPending { 
            shared_data: Arc::new(Mutex::new(SharedData { 
                completed: false, 
                waker: None 
            })) 
        };
        let shared_data_clone = Arc::clone(&task.shared_data);
        std::thread::spawn(move || {
            std::thread::sleep(std::time::Duration::from_millis(millis));
            let mut data = shared_data_clone.lock().unwrap();
            data.completed = true;
            if let Some(w) = data.waker.take() {
                w.wake();
            } 
        });
        task
   }
}

impl Future for SayHelloInPending {
    type Output = String;

    fn poll(self: std::pin::Pin<&mut Self>, cx: &mut std::task::Context<'_>) 
  				-> std::task::Poll<Self::Output> {
        let shared_data = self.shared_data.lock();
        if let Ok(mut data) = shared_data {
            if data.completed {
                return Poll::Ready("completed".to_string());
            } else {
                data.waker = Some(cx.waker().clone());
            }         
        }
        Poll::Pending
    }
}
```

可以看到，SayHelloInPending 就是我们的异步函数，现在换成了用 struct 实现，它的成员被放在了 SharedData 里面，并用 Arc 和 Mutex 保护起来，因为 Future 一来被 IO 线程读写，二来也会被主线程读写，因此需要这两个异步容器保护。SharedData 保护了 completed 字段，表示 IO 是否已经完成，初始值为 false。至于 waker 我们待会再说。

在 new 出 SayHelloInPending 的时候，我们启动了一个线程 sleep 了 n 毫秒用以模拟 IO 操作所需要的时间，可以看到，当 sleep 结束的时候，我们会调用 waker 的 wake 函数，这个名字看上去是要唤醒什么，实际上即触发我们之前提到的 sender 讲就绪的 Future 发给队列，等待 main 线程再次调用 poll 获得返回值。我们待会再说 waker 的更进一步细节。

之后的 impl Future for SayHelloInPending 块中，即实现了 Future trait，我们先不看 pin 和 context，就看里面的实现细节，可以看到，主要就是检查线程是否 completed，如果是，则返回 completed，否则返回 Pending状态。

这样我们就封装好了一个异步“函数” （Future）。



## 封装Task

那么主线程怎么调用上面封装的 Future 呢？这里封装了一个 Task，毕竟 Future 可以很多，但是我们需要抽象出 Task 方便 sender 发消息给到队列。

我们看看 Task 的代码：

```rust
struct Task<T> {
    fut: Mutex<Option<BoxFuture<'static, T>>>,
    sender: SyncSender<Arc<Task<T>>>,
}

impl<T> Task<T> {
    pub fn new(sender: SyncSender<Arc<Task<T>>>, f: BoxFuture<'static, T>) -> Self {
        Task {
            fut: Mutex::new(Some(f)),
            sender,
        }
    }
}

impl<T> ArcWake for Task<T> {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        arc_self.sender.send(arc_self.clone()).expect("failed to send task");
    }
}
```

 这里的返回值我用了模板实现，不必太在意。主要看的是，Task 首先有一个 fut 成员，也即我们的 Future，因为是多线程，因此需要 Mutex 保护，Box 也变成了 BoxFuture，这个我们以后说，现在只需要知道这个字段是 Future 即可，它要等待调度的。另一个当然是 sender，因为当 我们的 fut 就绪的时候，需要 sender 把这个 Task 发到队列，可以看到，sender 实际上放着的是 Arc\<Task\<T\>\>，因为 Task 在队列中，可能被其它线程共享，当然本例中只有两个线程（IO 线程和主线程），但我们还是习惯加上了 Arc 容器。

Task 的 new 方法接受一个 Future 和 sender，这当然是为了初始化。重点是 impl\<T\> ArcWake for Task\<T\> 。是的，这个 Task 就是上面提到的 waker 的实现，只要调用 waker 的 wake 方法，那么就会触发 wake_by_ref 的调用，在 wake_by_ref 里面，会看到， Task 的 sender 给队列发送了 Task。



## 总结 Future 和 Task

可见，Rust 的异步是这么实现的：

0、主线程不停的在监听队列，取出等待调用的 Future，调用 Future 试图获得这个“函数”的返回值；

1、但Future 实现了阻塞 IO 逻辑，当 IO 逻辑未完成时，若调用其 poll 方法，会返回 pending。主线程第一次调用发现是 Pending 返回，于是忽略继续处理下一个队列中的 Task；

2、当 Future 的 IO 逻辑完成后，状态会被改成就绪，此时调用 waker 的 wake 方法触发后续操作；

3、waker 其实就是 Task，Task 的 wake_by_ref 是被 waker 的 wake 方法触发的，其中会调用 sender 的 send 方法把 Task 放入队列；

4、主线程不停的在监听队列，又遇到了刚才的 Task，于是又去 poll 了一下，发现此时可以返回 Ready 了，于是拿到了“函数”的返回值。

画成图是这样：

![异步调用示例](https://www.jackhuang.cc/svg/rust-future-poll.svg)

可以看到，Future 在处理阻塞的 IO 逻辑时，是采用线程来完成的，因为不能阻塞主线程，主线程需要处理下一个 Future（Task）。还可以看到，Task 和 Future 是相互包含，引用的。

Task 实现了 Wake trait，放在 Future 中，当 Future 完成 IO 阻塞操作后，通过 Wake trait 调用 Task 的 wake_by_ref 函数激发 sender把 Task 重新放入队列中等待调度。

Future trait 同样也包含在 Task 中，因为 Task 被放入队列后会被 Receiver 调度，即调用其 poll 方法，获得 Pending 状态或者 Ready。Pending 状态则 Task 一边执行去，不阻塞 Receiver，而 Ready 状态则绑定了返回值。

![Task 和 Future 的关系](https://www.jackhuang.cc/svg/rust-future-task.svg)



## Context 出场

了解上面的关系后，现在该讲讲 Context 了。为什么 Future 在获取 Task，也即 waker 的时候没有直接去使用 waker 呢？这是因为 Context 封装了上下文，目前的确只有 waker，但这也为以后扩展留下了更好的封装。



## executor 驱动

现在，可以写我们的 executor 函数了，它无非就是把上面提到的receiver做的时候做一遍：

```rust
async fn excutor(queue: Receiver<Arc<Task<String>>>) {
    loop {
        match queue.recv() {
           Ok(task) => {
                let waker = waker_ref(&task);
                let contex = &mut Context::from_waker(&*waker);
                let mut fut = task.fut.lock().unwrap();
                if let Some(mut f) = fut.take() {
                    let result = f.as_mut().poll(contex);
                    if  result.is_pending() {
                        *fut = Some(f);
                    } else if let Poll::Ready(s) = result {
                        println!("finish and receive: {}", s);
                        break;
                    }
                }
           } 
           Err(_) =>  {

           }
        }
    }
}

fn main() {
    // 创建消息队列
    let (sender, queue) = sync_channel::<Arc<Task<String>>>(10);

  	// 创建任务，即我们的异步函数
    let task = Task::<String>::new(sender.clone(), SayHelloInPending::new(2 * 1000).boxed());

  	// 发送到队列，开始调度
    sender.send(Arc::new(task)).unwrap();
    
  	// 执行调度
    block_on(executor(queue));
}
```

可以看到，executor 执行的就是前面讲的内容，注意 Task 是如何转为 waker 的，以及 Context 对象的生成。



## 总结

以上可见，Rust 的异步编程是靠 Future 这个关键 trait 实现的，通过调用 poll 方法知道“函数”的状态，若是阻塞，则我们的主线程也即executor 会处理下一个 Task，若就绪，则获得函数的返回值。

我们日常编程当然不需要写这么复杂的代码，这里主要是在学习 Rust 的异步编程模型，实际上，Rust 的异步库都帮我们封装好了以上这些内容，我们在使用各个异步库的时候，只需要像前面一节课讲的那样，调用 Future 的 await 即可实现异步编程了。我们看看，如果把本节内容实现的东西写成 await 是什么样子，以下是伪代码：

```rust
    async fn SayHelloInPending() -> String {
        // ...
    }   
		let result = SayHelloInPending().await;
    println!("result = {}", result);
```

 就一行 await 即可。当执行到 await 时，由于 SayHelloInPending 需要等待 IO 完成，此时主线程不会阻塞在 await 这个地方，此时相当于调用了 Future 的 poll 方法，主线程发现是Pending于是跑去执行别的 aysnc 函数（即下一个 Task 的 Future）去了，等到 await 返回才又继续往下执行println（即这个 Task wake 方法调用了 sender 把 Task 放入队列继续给调度器执行）。这段代码后面的奥妙，正是本节讲的内容。



## 遗留问题

前面提到了一些我们直接忽略的东西，比如：Pin 这个类型，Pin 到底是什么，为什么 Future 放入 Box 的时候要变成 Pin\<Box\<T\>\>，也即 BoxFuture 呢？下一节我们继续前进吧！



 
