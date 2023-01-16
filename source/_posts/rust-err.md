---
title: Rust的错误处理
date: 2023-01-16 13:07:36
tags:
categories:
- Rust
---

Rust的错误处理主要有以下4点需要学习：

1、使用panic!宏；

2、使用Result\<T, E\>;

3、使用Optional\<T\>

4、使用 ? 符号简化错误处理的代码，这一点非常赞。

现在就看看我们Rust的错误处理具体细节吧。

<!--more-->

<!-- toc -->

## panic!宏

这个是最简单也是最暴力的错误解决处理方案，即直接中断程序运行，并且报告程序错误。如果程序无法继续往下执行，那么，尽早中断也是不错的选择，关键是，如果配合环境变量RUST_BACKTRACE=1的话，还可以打印从panic的地方开始的调用栈，方便快速定位问题：

```rust
fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("the denominator should not be 0");
    }
    a / b
}

fn main() {
    let s = 16;
    let t = 0;
    println!("s * t = {}", divide(s, t));
}
```

如果运行的时候加上环境变量RUST_BACKTRACE=1，则可以在检查分母的失败时打印出调用栈：

```rust
RUST_BACKTRACE=1 cargo run
```

 输出为：

```rust
thread 'main' panicked at 'the denominator should not be 0', src/main.rs:4:9
stack backtrace:
   0: _rust_begin_unwind
   1: core::panicking::panic_fmt
   2: greeting::divide
             at /Users/jack/Documents/code/rust/vsrust/greeting/src/main.rs:4:9
   3: greeting::main
             at /Users/jack/Documents/code/rust/vsrust/greeting/src/main.rs:12:28
   4: core::ops::function::FnOnce::call_once
             at /private/tmp/rust-20220812-6466-1atesch/rustc-1.63.0-src/library/core/src/ops/function.rs:248:5
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```



## 使用Result\<T, E\>

大部分错误都是可以容忍的，或者说我们可以继续保持程序继续运行，同时给用户一个友好提示，所以给一个恰当的提示是大多数场景。这个时候我们一般使用Result\<T, E\>。这是一个很简单但非常有用且常用的枚举类型，其源代码为：

```rust
pub enum Result<T, E> {
    /// Contains the success value
    #[lang = "Ok"]
    #[stable(feature = "rust1", since = "1.0.0")]
    Ok(#[stable(feature = "rust1", since = "1.0.0")] T),

    /// Contains the error value
    #[lang = "Err"]
    #[stable(feature = "rust1", since = "1.0.0")]
    Err(#[stable(feature = "rust1", since = "1.0.0")] E),
}
```

即Ok和Err两个枚举类型，很多标准函数都会用它来表示是否执行成功，例如，我们现在写一个程序，打开一个文件，如果该文件不存在，则创建它，否则就往文件里面写hello world：

```rust
use std::{fs::File, io::Write};

fn open_or_create_file() -> Result<(), std::io::Error> {
    let file_handle_result = File::open("rust_test.txt");
    match file_handle_result {
        Ok(mut file_hanle) => {
            file_hanle.write_all("你好世界".to_string().as_bytes());
            Ok(())
        },
        Err(error_message) => {
            let new_file_hanle_result = File::create("rust_test.txt");
            match new_file_hanle_result {
                Ok(mut file_hanle) => {
                    file_hanle.write_all("你好世界".to_string().as_bytes());
                    Ok(())
                },
                Err(error_message) => {
                    panic!("failed to create rust_test.txt, since: {}", error_message.to_string());
                }
            }
        }
    }
}

fn main() {
    let r = open_or_create_file();
    match r {
        Err(error_message) => {
            println!("something wrong: {}", error_message.to_string());
        },
        _ => (),
    }
}
```

洋洋洒洒写了很多，其实就是做了一个简单的逻辑，即先打开文件，存在则写入，不存在则创建，创建成功后写入，创建失败则panic。这个算是比较基本的Result\<T, E\>使用了。但显然，Rust不会让我们写这么多 match 或者 if 去做判断Result\<T, E\>结果的，下面就看看Rust是怎么把这段代码简化的。



## 用 ? 来简化返回的错误值处理

Rust提供了了 ? 号来帮助我们简化这种常见的Result\<T, E\>判断。? 其实就是一段简单的逻辑，即，如果返回的是Err那么就直接返回，如果是Ok则继续往下执行：

```rust
fn open_write() -> Result<(), std::io::Error> {
    File::open("rust_test.txt")?.write_all("你好世界".to_string().as_bytes());
    Ok(())
}

fn create_write() -> Result<(), std::io::Error> {
    File::create("rust_test.txt")?.write_all("你好世界".to_string().as_bytes());
    Ok(())
}

fn main() {
    let r = open_write();
    match r {
        Err(error_message) => {
            println!("something wrong: {}", error_message.to_string());
            create_write();
        },
        _ => (),
    }
}
```

可以看到，使用 ? 号后，简化了我们对Result\<T, E\>的判断处理，代码更简洁了。上面的代码中，open() 方法返回了Result\<T, E\>，? 会处理成功与失败的情况，若成功，则展开Ok(T)，也即获得文件对象句柄，进而调用write_all() 方法写入数据，若失败，则提前返回。

这里需要注意的是，open_write() 和 create_write() 返回值必须和里面的代码匹配，即成功返回的是Ok(())，失败则返回的是File::open() 或File::create() 的失败返回值Err(E)，此处E是std::io::Error。



## 使用Optional\<T\>也是一种选择

Optional\<T\>也可以像Result\<T, E\>那样用，如果是None那么就直接返回，否则就继续处理，比如前面的HashMap，我们get一个key，若存在，则乘以二返回，否则就返回None：

```rust
fn multiple_2(h: &HashMap<String, i32>, key: &str) -> Option<i32> {
    Some(h.get(key)? * 2)
}

fn main() {
    let mut h = HashMap::new();
    h.insert("hello".to_string(), 2);
    h.insert("rust".to_string(), 4);
    h.insert("jack".to_string(), 6);

    println!("the result is {:?}", multiple_2(&h, "jack"));
    println!("the result is {:?}", multiple_2(&h, "rose"));
}
```

可以看到，如果通过get() 方法访问一个HashMap存在的key，那么 ? 会展开Some(&i32)得到i32的对象的引用，并且和2相乘。如果不存在，则get() 方法返回None，函数提前返回，这段代码的输出是：

 ```rust
 the result is Some(12)
 the result is None
 ```



## 什么时候panic什么时候返回错误码

那么什么时候panic什么时候返回错误枚举呢？个人觉得这个问题还是比较显而易见的，从后端服务器设计角度来讲，进程退出是难以维护的不够健壮的提现，好的后端服务，应该能经受得住各种输入数据而屹立不倒，如果随随便便就退出以示对错误的一种反应，则容易被恶意客户端所利用，造成拒绝服务攻击。

也有人认为，出了错就尽快退出，这样可以及时发现错误，这个愿望是很好的，在程序调试阶段我们可以这么干，的确有利于调试，但放到线上，就需要深思熟虑了。

折衷方案是，对于后端服务初始化阶段，我们可以对关键信息做panic，运行时，对损害健壮性的请求进行提前拦截并提示错误。业务流程错误则当然使用错误信息。

再进一步说，对于出错，我们其实不能仅仅只是靠panic或者返回错误信息，需要更系统的去考虑问题，比如系统的吞吐（延迟），成功量（率），失败量（率），请求量，不同水平的告警，等等各种多维度的去监控才能准确的知道一个系统的健康与否。这里只是Rust语言学习，不展开说了。
