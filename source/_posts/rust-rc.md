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

