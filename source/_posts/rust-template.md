---
title: Rust的模版和trait编程
date: 2023-01-17 13:13:33
published: false
tags:
categories:
- Rust
---

要开始学习Rust比较高级的内容了。

今天要讲的是Rust的模板编程。模板编程实际上并不少见，不论C++还是Java，只要使用容器，基本都离不开模板。Rust的模板和C++很相似，都是编译时根据模板代码生成实际的（concrete）代码然后再编译成二进制。

那么，今天就好好看看Rust的模板编程吧。

<!--more-->

<!-- toc -->

## 模板

### 函数入参模板

假设我们需要做查处整型和字符型数组中，最大的那个元素，那么需要两个函数分别处理：

```rust
fn find_largest_int(array: &[i32]) -> &i32 {
    let mut largest = &array[0];
    for i in array {
        if largest < i {
            largest = i;
        }
    }
    largest
}

fn find_largest_char(array: &[char]) -> &char {
    let mut largest = &array[0];
    for i in array {
        if largest < i {
            largest = i;
        }
    }
    largest
}
```

很明显，两个函数都差不多，更抽象的写法是使用模板，即不管什么类型，都执行加法运算：



### 函数返回值模板

### struct模板

### 方法的模板



## trait编程

### 什么是trait

### 实现trait

### trait bond

