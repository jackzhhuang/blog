---
title: 无惧并发
date: 2023-01-31 17:46:41
published: false
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

主线程启动子线程后，会使用sender 调用 send 方法

### mutex\<T\> 与 Arc\<T\>



## 注意死锁



