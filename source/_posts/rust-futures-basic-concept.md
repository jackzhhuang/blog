---
title: Rust 异步编程（一）：基本概念和编程模型
date: 2023-02-10 23:41:58
tags:
categories:
- Rust
- Concurrency
---

Rust 的异步编程内容太庞大了，从最基础的 futures，到各种第三方库，都有很多内容可讲，今天开始，Rust 编程会聚集到异步编程上，当然，异步编程不仅仅是异步，实际也会涵盖比如网络编程等方面。现在，先来个最基础的开胃菜，基本概念和简单的异步编程。

<!--more-->

## 线程的上下文切换

现在的服务器操作系统都是多任务可打断式的，即多个任务同时在跑，任务之间会争夺CPU和内存资源。操作系统会有一套算法来调度这些任务，Linux 系统中最经典的就是公平调度算法，算法的思想很简单，就是每次总是运行获得CPU时间最小的那个任务。

那么什么时候触发这个调度算法呢？有两个时间点：

1、周期性的调度；即操作系统总是周期性的做一些事情，这个是不可打断的，比如周期性的更新系统时间，周期性的看看IO是否OK，当然也就包括周期性的执行任务的调度算法；

2、系统调用的时候；即用户态在调用系统调用的时候，会陷入内核态，内核态除了完成用户态的请求，还会执行一次任务调度。

周期性的执行是不可打断的事情，属于实时调度，这方面我们的进程作为用户态的程序，一般没有机会去优化。

但系统调用却是靠我们用户态的程序来触发的。所谓系统调用，就是调用操作系统的原生接口，比如sleep，poll等，也就是说，如果我们的程序调用系统调用，会触发一些列的操作系统操作：

1、切换上下文到内核态；

2、完成用户态请求；

3、进行任务调度；

4、切换上下文回到用户态。

可能还有其它更多的事情，因此，系统调用代价是很大的。如果有两个线程，那么这两个线程的争夺将触发一系列的代价。



## IO密集型和CPU密集型任务

我们从系统资源的角度去看线程任务的话，那么主要分为两种：IO 密集型任何和CPU 密集型任务。前者主要会有大量的 IO 操作，因为需要 IO，因此会产生大量的系统调用，比如 read，write，accept 等等，按前面所说，这里将有巨大的上下文切换代价；后者则主要占用 CPU 资源，几乎没有系统调用，但会有大量的应用代码，比如 for， while ，loop，以及对 cache 对内存的频繁访问。

因此，分离 IO 密集型的任务和 CPU 密集型任务到不同的线程，就有助于提高线程的执行效率，比如一个请求，需要算一算，再读一下DB，最后再算一算，再返回给客户端，一共四个步骤，如果我们按照 IO 密集型和 CPU 密集型任务分离，那么就是两类操作，即读 DB 和返回客户端是 IO密集型任务，算一算则是 CPU 密集型任务，我们的思路是：遇到 IO 任务，则挂起，去做别的事情，保证我们的工作线程跑满 CPU，这样分离后，CPU 利用率就能极大提高，而且，因为遇到 IO 任务，我们去做别的事情不是线程切换，而是应用层的一种调度，这样减少了上下文的切换，减少了系统调度的代价。



## Rust 的异步trait库——futures

futures 库提供了最基本的上面说的任务模型最基本的trait，官方文档中给了一个比较复杂的例子（其实不复杂，但对于最初的学习者来说，可能会遇到大量的信息应付不过来，工程上，已经有很成熟的 tokio 库实现，这个库以后再讲，我们先把底层原理学清楚），我这里打算从一个更简单的例子开始慢慢演化成 Rust 官方文档案例的样子，有助于理解。

其实，Rust 的 futures 库并不完整，更多是在定义一个 trait，虽然给了一些实现宏，但还是留有大量的空间给我们去实现，好在第三方库tokio 给出了很好的实现，但正如官方文档所说，你的确可以直接去学习那些库的用法，但最后总是要回过头来了解，futures 库做了什么，或者说定义了什么，Rust 期待的异步模型是什么样子的，这些底层原理还是要搞懂的。

不像官方文档直接拿出futures trait做例子，这一节，暂时不细讲 futures trait 都定义了什么，我们先把最基础的异步模型讲清楚，再引入 futures trait ，以及其它更多概念，这样，我们先引入一个基本模型，然后再逐个引入各种概念（比如futures trait）从而理解引入的概念解决了什么问题，或者用于做什么的，这样比官方文档直接把大量概念加入进来更容易理解。

为了更简单更容易理解，我的例子甚至不使用网络库，文件库这些 IO 库，只留下最精简的模型代码，这样我们可以完全聚焦在异步编程模型上，后面我们再加入 IO 库，从而理解 IO 任务和 CPU 任务分离模型基础。

当我们把这些事情都完全展现出来后，再去看 tokio 库，就有了然于胸的感觉了。



## 生产者和消费者模型

异步编程最经典的模型就是生产者和消费者模型，它的原理非常简单：即生产者生产消息，消息放入队列，队列的消息先进先出，因此消费者在队列另一头读出消息然后处理。把这个模型套到上面提到的 IO 任务和 CPU 任务上，那就是，生产者是 IO 任务，而 消费者是 CPU 任务，中间用队列通信，当 CPU 任务遇到 IO 时因为要被阻塞了，因此，会把任务放回队列中，待下次调度，自己会去做其它的CPU任务，用一张简单的图画出来就是这个样子：

![基本生产者消费者模型](https://www.jackhuang.cc/svg/basic-async-task.svg)

这个模型中，除了 IO 线程会陷入内核态（因为需要调用系统的 IO 接口，属于系统调用），其它地方一般不会再有系统调用，至少不会再有明显阻塞式的 IO 系统调用，保证了 CPU 线程再各个任务间切换时，没有系统调用产生的上下文切换代价。

那么我们就开始实现这个最基本的模型吧。当然我这里不是实现一个通用的生产者消费者异步处理库，因此代码会比较偏硬，属于教学代码，我们重点放在模型的理解，以及后续各个概念引入时我们可以很方便的理解这些东西的作用，或解决了什么问题。



### Rust 的消息队列模型

Rust 提供了原生的消息队列模式，这个在之前提到过：

https://www.jackhuang.cc/2023/01/31/rust-multiple-thread/#channel

这里我们打算使用 sync_channel。sync_channel 和 channel 都是用于实现生产者消费者模型的，不同之处是：

1、channel 的队列是“无限”大的，当然不可能是“无限大”，因此为了限制队列的大小，应该使用 sync_channel，它接受一个参数，若队列满了，那么 send 会被阻塞；由于 channel 的队列 “无限大”，因此理论上 send 是不会被阻塞的。

2、而sync_channel 维护了一个 buffer 队列，接受一个参数控制队列大小。参数可以是 0，当参数为 0 的情况下，send 会被阻塞，知道receive 取走消息。因此，sync_channel 的send 是会被阻塞的。

我们实现生产者消费者模型，就是利用 sync_channel 来为我们维护中间的消息队列。



### async 函数和 wait调用

这里，我们开始引入第一个 Rust 异步编程概念，即 async 函数。函数这个概念我们很熟悉了，函数前面加上 async 关键字相当于表示，这个函数会被异步调度执行。

记住，是异步调度执行，即：函数不会马上执行，是由我们调度后才能执行，举个简单的例子：

```rust
async fn print_helloworld() -> String {
    println!("hello world!");
    String::from("hello world!")
}

fn main() {
    block_on(async {
        let fut = print_helloworld();
        let value = fut.await;
        println!("return: {}", value);
    });
}
```

我们先看第一行，是一个函数，打印 hello world 后返回一个 hello world 字符串，和之前不同的是，函数前面有一个 关键字 async。先暂时不管它，往下看，第 7 行，来了一个 block_on 调用，这是因为所有的异步编程都必须用异步的方式调用，而main 函数是同步函数，那么异步和同步函数之间，必须用像 block_on 这样的同步函数连接，实际上 block_on 帮我们做了一件简单的事情：即等待异步调用结束，避免异步调用还未结束 main 函数就退出了，这样进程就没有了。

block_on 中有一个 async 块，表示里面的代码将会是异步执行的，可以看到，里面调用了 print_helloworld，但实际上，因为是异步执行，此时 print_helloworld 并没有执行，至于什么时候触发呢？就是第 9 行的 fut.await。

但 print_helloworld 不是返回一个 String 吗？String 没有 await 字段。这就是 async 的作用，当函数前被加上 async 后，函数的返回值会变成—— future，future 封装了返回值和一些特性，但之前说了，这一节我们暂时不深入到 future，这一节，我们关键去理解生产者消费者模型，此处，我们只需要认为， async 的 出现，使得函数返回值变成了future，而 future 封装了函数的返回值就行。

future 提供了一个 await 调用，我们一旦调用 await 函数就会被触发执行，正如第 9 行那样，它将会触发 print_helloworld 的执行。await 的返回值就是函数的返回值。

网上看到很多疑问，就是这并不是异步执行啊，只是单个线程延迟执行了函数而已，类似闭包。

的确如此，我们并没有开启多线程，一个线程怎么可能会异步执行呢？Rust 没有这种魔力。所以这里我就要提醒一下我已经说的话：

1、Rust 没有完全实现异步调用模型，它留下了很多空白，tokio 库补上了；这里的 asyn 和 await 只是提供了能异步调用某个函数的能力，并不是实现了异步调用。

2、想异步执行函数，需要生产者消费者模型，async 和 await 只是给生产者和消费者模型提供了语言上的工具，剩下的我们还是需要自己去实现，当然，tokio 已经为我们实现很多常用场景了，我们后面再说，现在我们需要做的是：理解 Rust 提供的 async 和 await 做了什么即可，后面我们将会看到，它们在生产者消费者这个异步模型中的作用。

运行上面的代码，程序会按照调用顺序打印出两个hello world：

```rust
hello world!
return: hello world!
```



### 任务

有了消息队列和 async/await 特性，我们可以去实现生产者消费者模型了。其思路是：创建一个任务类Task，Task 封装 上面提到的future，因此，我们的生产者生产 Task 对象，放入队列，消费者从队列取出 Task 对象，然后调用 await 执行，就是这么简单，往后我们会把这个模型引入更多概念和做法，最终理解整个 Rust 异步编程都有什么内容，现在，就从这个简单的模型开始吧。

我们先说一下我们的任务：

```rust
struct Task<T> {
    fut: Mutex<BoxFuture<'static, T>>,
}

impl<T> Task<T> {
    pub fn new(f: BoxFuture<'static, T>) -> Self {
        Task {
            fut: Mutex::new(f),
        }
    }
}
```

这里，任务只有一个字段 fut，即我们之前提到过的 future，它有await字段可以触发函数的执行。其中，Mutex 我们已经很熟悉，而 BoxFuture\<T\> 是 Pin\<Box\<T\>\>  的 type 别名。Box 我们也很熟悉了，但 Pin 是什么呢？现在我们简单理解 Pin 是帮我们实现内存拷贝的一个辅助结构，后面我会提出一个问题，并用 Pin 解决它，那个时候就很好理解了，总之我们现在还是先聚焦在生产者消费者模型吧。

总之，Task 是一个任务，它的 fut 保存着 async 函数，在 CPU 线程中等待 await 的调用。我们的生产者会生产 Task 对象，即调用上面的 new 方法，然后 send 到队列中，等待消费者调用 await 实现异步。泛型 T 是函数的返回值，用于限定 let x = fut.await; 中的 x 类型的。

 

### 将要被异步调用的函数

```rust
async fn say_hello() -> String {
    println!("hello");
    String::from("hello")
}

async fn say_world() -> String {
    println!("world");
    String::from("world")
}

async fn say_bye() -> String {
    println!("bye");
    String::from("bye")
}
```

我们写出三个简单的将要被异步调用的函数，函数非常简单，没有 IO ，还是那句话，我们本节聚焦生产者消费者模型的实现。

这三个函数将会返回 future，他们将会被传入三个 Task 对象中去，然后进入队列，等待消费者线程异步拉起 await 调用。



### 执行器

```rust
async fn excutor(queue: Receiver<Task<String>>) {
    let mut count = 0;
    loop {
        match queue.recv() {
           Ok(task) => {
                let r = task.fut.lock().unwrap().as_mut().await;
                println!("await return: {}", r);
                count += 1;
                if count == 3 {
                    return ;
                }
           } 
           Err(_) =>  {

           }
        }
    }
}
```

执行器是我们这次又引入的一个概念，它的任务就是执行消费者要做的事情，甚至可以起名叫消费者都可以，但有些框架，会在执行器中spawn 出新的线程去跑逻辑，逻辑被 IO 阻塞的时候才返回给执行器并通知 IO 线程处理，等待任务重新进入队列被调度。

我们这次这个例子很简单，不去起新的线程，直接在执行器中从消息中取出任务，拿到 future 对象后调用 await 执行函数，当三个函数都被执行后则退出。

可以看到执行器函数也是 async 的，之前也说了，async 函数才能调用 另一个asyn 函数，因为我们饿 future 已经是 async的了，所以我们的执行器也必须是 async 的。



### main函数的实现

```rust
fn main() {
    // 创建消息队列
    let (sender, queue) = sync_channel::<Task<String>>(10);

    // 创建三个即将被异步调用的函数，其返回 future
    let hello_fut = say_hello();
    let world_fut = say_world();
    let bye_fut = say_bye();

    // 创建三个Task，存放之前三个函数的future
    let t1 = Task::new(hello_fut.boxed());
    let t2 = Task::new(world_fut.boxed());
    let t3 = Task::new(bye_fut.boxed());

    // 以下创建三个生产者线程，用以模拟多个IO生产的数据操作
    // 第一个生产者线程
    let sender1 = sender.clone();
    thread::spawn(move || {
        sender1.send(t1).unwrap();
    }); 

    // 第二个生产者线程
    let sender2 = sender.clone();
    thread::spawn(move || {
        sender2.send(t2).unwrap();
    }); 

    // 第三个生产者线程
    let sender3 = sender.clone();
    thread::spawn(move || {
        sender3.send(t3).unwrap();
    }); 

    // 执行消费者工作，从队列中取出Task，触发await
    block_on(excutor(queue));
}
```

上面就是 main 函数做的事情，注释已经写得很清楚，我们实现了一个简单的生产者消费者模型，这是最简单最基本的异步编程模型，利用了 Rust 以下特性：

1、消息队列；

2、async/await 调用；

3、future trait；

4、线程函数。

这个模型非常简单，也因此，其实有很多地方可以讨论，有很多问题待解决，因此我们在后续的文章中会对这个模型提出一些问题，从而引入更多细节的东西解决这些问题，最终理解 Rust 异步编程。

下一节，我们首先讨论是：future 到底是什么？我们将聚焦 future，看看 future在这个模型中扮演的角色，解决了什么问题。
