---
title: Rust的Rc容器
date: 2023-01-29 10:14:33
published: false
tags:
- Rust
- Smart Pointer
categories:
- Rust
---

前面讲了 Box\<T\> ，和一般的变量一样，它只能有一个所有权方，如果需要有多个所有权，即实现类似（不完全一样，因为Rust只要你不说，都是不可变的）python那样的所有权复制，就需要Rc\<T\>了。

<!--more-->

之前的Box\<T\>实现了一个如同以下的链表：

![Box\<T\>实现无限链表](https://www.jackhuang.cc/svg/conforbox.svg)

其代码如下：

```rust
#[derive(Debug)]
enum List {
    Con(i32, Box<List>),
    Nil,
}

use List::Con;
use List::Nil;

fn main() {
    let a = Con(1, Box::new(Con(2, Box::new(Con(3, Box::new(Nil))))));
    println!("a = {:?}", a);
}
```

如果串入另一个链表如下，Box\<T\>就不能实现了，因为Box\<T\>是只能有一个所有权方的：

