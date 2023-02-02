---
title: 无惧并发
date: 2023-01-31 17:46:41
tags:
- multiple-thread
- Rust
categories:
- Rust
---

前面所有的例子，都是单线程的，现在我们来看看 Rust 都提供了什么多线程能力。

<!--more-->

## 基本操作

### 启动线程

使用 thread 库的 spawn 方法生成一个 thread，其入参是一个closure：

```rust
    thread::spawn(|| {
        for i in 1..10 {
            println!("in thread: {}", i);
            thread::sleep(Duration::from_millis(1000));
        }
    });

    for i in 1..5 {
        println!("in main: {}", i);
    }
```

输出如下：

```rust
in main: 1
in main: 2
in main: 3
in main: 4
in thread: 1
```

可以看到，main 线程结束没等子线程运行完就直接整个线程退出了。



### 等待线程结束

所以，我们需要在主线程 join 子线程：

```rust
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("in thread: {}", i);
            thread::sleep(Duration::from_millis(1000));
        }
    });

    for i in 1..5 {
        println!("in main: {}", i);
    }

    handle.join().unwrap();
```

 此时可以保证子线程结束的时候主线程才退出，因为 join 会挂起，直到子线程完成：

```rust
in main: 1
in main: 2
in main: 3
in main: 4
in thread: 1
in thread: 2
in thread: 3
in thread: 4
in thread: 5
in thread: 6
in thread: 7
in thread: 8
in thread: 9
```



### 子线程panic

子线程 panic 是不会影响主线程，也不会影响其它线程：

```rust
    let handle = thread::spawn(|| {
        for i in 1..5 {
            println!("in thread: {}", i);
            thread::sleep(Duration::from_millis(300));
            panic!("thread panic!");
        }
    });

    let handle2 = thread::spawn(|| {
        for i in 1..5 {
            println!("in thread2: {}", i);
            thread::sleep(Duration::from_millis(1000));
        }
    });

    for i in 1..5 {
        thread::sleep(Duration::from_millis(1000));
        println!("in main: {}", i);
    }

    handle.join();
    handle2.join();
```

输出如下：

```rust
in thread: 1
in thread2: 1
thread '<unnamed>' panicked at 'thread panic!', src/main.rs:35:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
in thread2: 2
in main: 1
in thread2: 3
in main: 2
in thread2: 4
in main: 3
in main: 4
```

可以看到，尽管thread 线程 panic了，但 thread2 和 main 线程并没有停止，依然继续打完了所有数字。

### 主线程panic

主线程的 panic 则会打断所有子线程：

```rust
   let handle = thread::spawn(|| {
        for i in 1..5 {
            println!("in thread: {}", i);
            thread::sleep(Duration::from_millis(1000));
        }
    });

    let handle2 = thread::spawn(|| {
        for i in 1..5 {
            println!("in thread2: {}", i);
            thread::sleep(Duration::from_millis(1000));
        }
    });

    for i in 1..5 {
        thread::sleep(Duration::from_millis(300));
        println!("in main: {}", i);
        panic!("main panic!");
    }

    handle.join();
    handle2.join();
```

输出如下：

```rust
in thread: 1
in thread2: 1
in main: 1
thread 'main' panicked at 'main panic!', src/main.rs:48:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

可见，main 线程 panic 会中断所有的线程。



### panic 与析构

既然 main 线程panic会中断所有线程，那么会影响析构吗？答案是：

1、若果是 main 线程的对象，不会影响 main 线程的对象析构；

2、若是子线程的对象，那么子线程的对象析构不会被触发。

先看看主线程panic的例子：

```rust
struct Person {
    age: i32,
    name: String,
}

impl Drop for Person {
    fn drop(&mut self) {
        println!("person is being destroyed");
    }
}

fn main() {
    let handle = thread::spawn(|| {
        let p = Person {
            age: 99,
            name: "jack".to_string(),
        };
        for i in 1..5 {
            println!("in thread: {}", i);
            thread::sleep(Duration::from_millis(1000));
        }
    });

    for i in 1..5 {
        thread::sleep(Duration::from_millis(300));
        println!("in main: {}", i);
        panic!("main panic!");
    }

    handle.join();
}
```

输出如下：

```rust
in thread: 1
in main: 1
thread 'main' panicked at 'main panic!', src/main.rs:45:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

可见子线程中 Person 的析构没有被调用。但如果 Person 对象是在主线程（不再贴出实现 Drop trait 的代码）：

```rust
fn main() {
    let p = Person {
        age: 99,
        name: "jack".to_string(),
    };

    let handle = thread::spawn(|| {
        for i in 1..5 {
            println!("in thread: {}", i);
            thread::sleep(Duration::from_millis(1000));
        }
    });

    for i in 1..5 {
        thread::sleep(Duration::from_millis(300));
        println!("in main: {}", i);
        panic!("main panic!");
    }

    handle.join();
}
```

输出如下：

```rust
in thread: 1
in main: 1
thread 'main' panicked at 'main panic!', src/main.rs:46:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
person is being destroyed
```

可见，Person 对象的析构被调用，主线程的对象析构没有受到影响。

那么，如果是子线程 panic 呢？比如子线程 panic，对象也在子线程上：

```rust
fn main() {
    let handle = thread::spawn(|| {
        let p = Person {
            age: 99,
            name: "jack".to_string(),
        };

        for i in 1..5 {
            println!("in thread: {}", i);
            thread::sleep(Duration::from_millis(300));
            panic!("thread panic!");
        }
    });

    for i in 1..5 {
        thread::sleep(Duration::from_millis(1000));
        println!("in main: {}", i);
    }

    handle.join();
}
```

输出如下：

```rust
in thread: 1
thread '<unnamed>' panicked at 'thread panic!', src/main.rs:40:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
person is being destroyed
in main: 1
in main: 2
in main: 3
in main: 4
```

可见，子线程 panic，其拥有的对象还是会及时析构。可见 panic 对析构的影响如下：

|                 | 对象在主线程 | 对象在子线程                            |
| --------------- | ------------ | --------------------------------------- |
| **主线程panic** | 对象析构     | <font color="red">对象不会被析构</font> |
| **子线程panic** | 对象析构     | 对象析构                                |



### move 所有权到子线程

之前在讲 closure 的时候，有说过：闭包只是引用上下文，除非使用 move 否则闭包不会获得上下文对象的所有权。我们启动一个线程的时候，也是使用闭包，但 Rust 的线程闭包如果要 catch 上下文，就必须 move：

```rust
fn main() {
    let p = Person {
        age: 99,
        name: "jack".to_string(),
    };

    let m = || {
        println!("peson in m = {:?}", p);
    };
    m();

    let handle = thread::spawn(|| {
        println!("person = {:?}", p);
    });

    handle.join();
}
```

上面的代码中，闭包 m 不会夺走 p 的所有权，因为它的定义前面没有 move，这样没问题。但线程的闭包却不行，线程的闭包必须获得上下文的所有权（如果线程有使用的话）。因为线程 catch 上下文对象，如果不获得所有权，那么可能会造成悬空指针的问题，因此上面的代码编译报错如下：

```rust
 --> src/main.rs:36:32
   |
36 |     let handle = thread::spawn(|| {
   |                                ^^ may outlive borrowed value `p`
37 |         println!("person = {:?}", p);
   |                                   - `p` is borrowed here
   |
note: function requires argument type to outlive `'static`
  --> src/main.rs:36:18
   |
36 |       let handle = thread::spawn(|| {
   |  __________________^
37 | |         println!("person = {:?}", p);
38 | |     });
   | |______^
help: to force the closure to take ownership of `p` (and any other referenced variables), use the `move` keyword
   |
36 |     let handle = thread::spawn(move || {
   |                                ++++
```

我们按照编译起的提示，在线程闭包前加上 move 即可正常运行。注意，此时 p 已经失去所有权，主线程后续不能再使用 p。



## 信息传递的两种主要方式

### channel

使用 move 方式 显然不是最好的，按 Go 语言的哲学：“Do not communicate by sharing memory; instead, share memory by communicating”。即用复制传递信息，而不是共享的方式传递。 Rust 的 sync::mpsc 库提供了 channel 方式来实现线程直接的消息通讯。mpsc 即 multiple producer single consumer，多生产者单消费者。

```rust
fn main() {
    let p = Person {
        age: 99,
        name: "jack".to_string(),
    };

    let (sender, recv) = sync::mpsc::channel();

    let handle = std::thread::spawn(move || {
        let p = recv.recv().unwrap();
        println!("receive a person: {:?}", p);
    });

    sender.send(p).unwrap();

    handle.join().unwrap();
}
```

上面的代码，首先第 7 行生成了一对 channel tuple，一个是 消息发送者sender，和接受者 recv，然后第 9 行 我们将 recv move 进（因为子线程中使用了recv，但没有使用 sender，因此 sender 还在主线程中）子线程，在子线程中，我们调用 recv 来拉取 sender 给过来的对象。

注意，sender 是在第 14 行，因此，子线程的 recv 是会阻塞的，直到主线程的 sender 发消息过来。recv 方法返回的是 Result，如果没问题，我们将会在子线程收到 Person 对象，否则则是Err。

主线程启动子线程后，会使用sender 调用 send 方法将 Person 传递给子线程，此时 recv 会结束阻塞，返回Result，若成功，unwrap 出来则是Person对象。

注意，Person 对象 send给子线程后，主线程就失去所有权了。

子线程若不想阻塞，可以调用 try_recv，如果子线程需要周期性的处理一些事情，可以写个循环，try_recv 一次，然后再处理一下其它事情，可以用多 sender 写个例子。



### 多sender

对于 sender，我们可以简单的调用 clone 来获得更多的生产者：

```rust
fn main() {
    let (sender, recv) = sync::mpsc::channel();

    let handle = std::thread::spawn(move || {
        loop {
            match recv.try_recv() {
                Ok(message) => {
                    println!("receive messge: {:?}", message);
                },
                TryRecvError => {

                }
            }
            std::thread::sleep(std::time::Duration::from_millis(500));
        }
    });

    let message = vec!["hello".to_string(), "world".to_string(), "!".to_string()];

    for m in message {
        let new_sender = sender.clone();
        new_sender.send(m).unwrap();
    }

    handle.join().unwrap();
}
```

上面的第 21 行就是 sender 通过 clone 方法直接创建了多个生产者。接受者则使用了 try_recv 来拉取消息。



### Mutex\<T\> 与 Arc\<T\>

前面一节讲过，Rc 和 RefCell 是不能用于多线程的，如果有一个资源需要多线程共享，那么我们就需要 Arc\<T\> 来进行引用计数，因为它的引用计数是原子的，即 atomic reference count。Arc\<T\> 和 Rc\<T\> 是一样的，只是它还能支持多线程使用。

同样，因为多个线程要修改这个资源，那么就需要 Mutex\<T\> 去代替 RefCell<T\>。Mutex\<T\> 同样和  RefCell<T\> 一样，但还能用于多线程。

因此，若想实现多个线程同时修改一个资源，那么就需要 Arc\<Mutex\<T\>\> 。

```rust
use std::{thread, sync::{self, mpsc::{Sender, RecvError, TryRecvError}, Arc, Mutex}};

#[derive(Debug)]
struct Person {
    score: u32,
    name: String,
    title: Option<String>,
}

fn main() {
    let boy = Arc::new(Mutex::new(Person {
        score: 0,
        name: "jack".to_string(),
        title: None,
    }));

    let score_arc = Arc::clone(&boy);
    let score_handle = std::thread::spawn(move || {
        loop {
            let mut p = score_arc.lock().expect("no person!");
            println!("add the score");
            if p.score == 8 {
                println!("score is up to 8");
                break;
            }
            p.score += 1;
            println!("one year pass");
            std::thread::sleep(std::time::Duration::from_millis(500));
        }
    });

    let title_arc = Arc::clone(&boy);
    let title_handle = std::thread::spawn(move || {
        loop {
            let mut p = title_arc.lock().expect("no person!");
            println!("check the score");
            if p.score == 8 {
                p.title = Some(String::from("PhD"));
                break;
            }
            std::thread::sleep(std::time::Duration::from_millis(100));
        }
    });

    loop {
        if let Some(title) = &boy.lock().unwrap().title {
            println!("the boy get title: {}",title);
            break;
        }
    }
}
```

上面的代码中，score_handle 这个线程不停的加分数，title_handle 这个线程不停的检查分数是否达标，若达标，则授予PhD，而主线程则不停的检查 title。

三个线程都在争夺 Person 对象资源，因此，Person 对象资源放在Mutex中。同时，多个线程需要访问 Mutex，需要对 Mutex 进行引用计数，因此，Mutex 放在 Arc 中。

其中的 lock 函数将会获得资源，若资源被占用，则线程 阻塞在 lock 函数。lock 函数返回 LockGuard，当超出作用域（即每次 loop 循环）则自动释放 lock，因此，三个线程都在争夺 Person 资源，从输出可以看出的确如此：

```rust
add the score
one year pass
add the score
one year pass
add the score
one year pass
add the score
one year pass
add the score
one year pass
add the score
one year pass
add the score
one year pass
add the score
one year pass
add the score
score is up to 8
check the score
the boy get title: PhD
```



## 注意死锁

尽管 Rust 已经做了很多防止悬挂指针的操作，但依然还是无法防止运行时的错误，死锁就是这样，因为这个不可能通过编译时就能找出来。

最常见的死锁就是两个线程户等对象已经持有的锁，比如 A 线程拿着 A 锁等 B 线程释放 B 锁，而 B 线程拿着 B锁等 A 线程释放 A 锁，这样A 和B永远无法解开。

防止这种死锁发生的关键是，每次使用锁的时候，应该多考虑以下问题；

1、是否必须加锁不可；实际上，胡乱加锁是出现死锁的重要原因，所以每次想到要加锁，那么就要好好思考：是否有不需要加锁的设计或者现有的锁已经可以满足需求；加锁并不是一件很酷的事情，实际上不到万不得已，就不去加锁。

2、如果非要加锁，那么对同一个资源的加锁操作需要保证顺序一致，互等死锁的问题就在于两个线程加锁顺序不一致造成的；也可以理一下，为什么访问一个资源要加两个锁？这种设计是否已经出问题？

3、锁的粒度是否过细；过细的锁很容易造成死锁，当然过粗的锁也会降低并行效率；可以回到第一步好好考虑是否应该加锁。

4、针对业务逻辑建立标准的生产者消费者模型或者规范可以很好的防止死锁发生。因为模型或者规范一旦建立，我们只需要集中写逻辑就行了，无需再对资源争夺问题想太多，少写危险代码就少点风险。底层的逻辑一旦建立好，就应该依赖或者复用，防止反复踩坑。



## Send  和 Sync trait

Send trait 可以让类型获得可以 send 的能力，Rust 几乎所有的类型都是有 send trait 的除了 Rc\<T\>。

Sync trait 则可以让类型获得多线程访问的能力，同样， Rc\<T\> 也是不能用于多线程的，RefCell\<T\> 也是，必须用 Mutex\<T\>代替。

一般情况下不会去手动实现 Send  和 Sync trait，因为它们是 unsafe 的，而且大部分类型都已经是有这两个 trait了，用它们组合而成的 类型也会带有 Send  和 Sync trait。



## 注意 join 方法

spawn生成一个线程后，其返回值是一个 JoinHandle，我们调用它的 join 方法等待线程结束，需要留意的是，join 方法的入参是 self 而不是&self，也就是说，join 后 handle 会失效。这就会有可能造成悬空指针。

可见，join 一般是等待线程结束时使用，之后就不应该再去操作线程了。



## 多线程下 trait 的 'static 属性

之前的 move 会把所有权移入线程中：

```rust
#[derive(Debug)]
struct Person {
    score: u32,
    name: String,
}

fn main() {
    let p = Person {
        score: 100,
        name: "jack".to_string(),
    };
    let handle = std::thread::spawn(move || {
        println!("p = {:?}", p);
    });

    handle.join().unwrap();
}
```

 move会把 p 所有权移入线程，但如果改成移入一个引用就会报错了：

```rust
    let p = Person {
        score: 100,
        name: "jack".to_string(),
    };
    let a = &p;
    let handle = std::thread::spawn(move || {
        println!("a = {:?}", a);
    });

    handle.join().unwrap();
```

编译结果如下：

```rust
error[E0597]: `p` does not live long enough
  --> src/main.rs:14:13
   |
14 |       let a = &p;
   |               ^^ borrowed value does not live long enough
15 |       let handle = std::thread::spawn(move || {
   |  __________________-
16 | |         println!("a = {:?}", a);
17 | |     });
   | |______- argument requires that `p` is borrowed for `'static`
...
20 |   }
   |   - `p` dropped here while still borrowed
```

 这个很容易理解，主线程可能已经析构 p，但 a 引用还在，此时就是典型的悬挂指针了。因此这里是不应该允许 move 引用的。改成 template 也会出现同样的问题：

```rust
#[derive(Debug)]
struct Person {
    score: u32,
    name: String,
}

fn some_print<T, F>(f: F) 
        where T: std::fmt::Debug,
              F: Fn() -> T {
    let a = f();
    let handle = std::thread::spawn(move || {
        println!("a = {:?}", a);
    });

    handle.join().unwrap();
}

fn main() {
    some_print(|| {
        Person {
            score: 100,
            name: "jack".to_string(),
        }
    });
}
```

编译上面的代码会有以下错误：

```rust
error[E0277]: `T` cannot be sent between threads safely
  --> src/main.rs:13:18
   |
13 |       let handle = std::thread::spawn(move || {
   |  __________________^^^^^^^^^^^^^^^^^^_-
   | |                  |
   | |                  `T` cannot be sent between threads safely
14 | |         println!("a = {:?}", a);
15 | |     });
   | |_____- within this `[closure@src/main.rs:13:37: 15:6]`
   |
note: required because it's used within this closure
  --> src/main.rs:13:37
   |
13 |       let handle = std::thread::spawn(move || {
   |  _____________________________________^
14 | |         println!("a = {:?}", a);
15 | |     });
   | |_____^
note: required by a bound in `spawn`
help: consider further restricting this bound
   |
10 |         where T: std::fmt::Debug + std::marker::Send,
   |                                  +++++++++++++++++++
```

这里的问题就是出在我们打算把 a move 进线程中时，Rust 需要我们保证 a 有以下能力：

1、可以 move 的能力，这个需要Send trait；（详见：https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/std/marker/trait.Send.html ）

2、保证 a 的生命周期覆盖整个线程生命周期。这个是似乎是显而易见的，但为什么呢？因为 Rust 怕我们把引用给 move 进去了，就好像前一个例子，a 是 p 的引用，此时 move a 到线程中是会出现悬空指针的。解决方案是向 Rust 保证：move a 到线程后，它的生命周期会完全覆盖线程的生命周期，使用 'static 生命周期即可。（这里和引用参数中的 'static 不一样，引用参数的 'static 是说这个引用是固化在常量存储区中的，它随进程的生命周期一起开始和结束）。修改方法如下：

```rust
fn some_print<T, F>(f: F) 
        where T: std::fmt::Debug + Send + 'static,
              F: Fn() -> T {
    let a = f();
    let handle = std::thread::spawn(move || {
        println!("a = {:?}", a);
    });

    handle.join().unwrap();
}
```

