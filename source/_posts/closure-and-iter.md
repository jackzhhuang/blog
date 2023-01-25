---
title: 闭包和迭代器
date: 2023-01-25 14:44:32
published: false
tags:
- Rust
- Closure
- Iterator
categories:
- Rust
---

闭包和迭代器很多语言都有，Rust也不例外，并且，数量掌握闭包和迭代器是写出一手好代码的必要因素。今天就把这两个概念拿出来说说。

<!--more-->

## 闭包

闭包，即closure，简单说即：

1、匿名函数；

2、可以当作value赋值，灵活调用；

3、闭包的入参和出参除了定义闭包的时候可以指定，还可以由编译器推断（这样更简洁）。

以下是最简单的例子：

```rust
fn main() {
    let v: Vec<String> = vec!["hello world!".to_string(), 
                              "hello rust!".to_string(),
                              "hello jack!".to_string(),];

    v.into_iter().for_each(|x| println!("print in closure: {}", x));
}
```

 这里的闭包即“|x| println!("print in closure: {}", x)”，这是一个匿名函数，入参为x，编译器会推理为String，根据代码的实现，没有返回参数，只是简单的打印了String的值。

关于闭包这里不在对其定义阐述更多基本的内容，只需要知道，Rust的闭包入参和出参都是可以依靠编译器推断的就行，关键是Rust的闭包相对于其它语言需要留意的地方。



### 是move还是引用？

闭包最大的特点就是可以capture上下文变量，那么，这个capture在Rust中是move还是引用呢？答案是：引用。例如：

```rust
    let list = vec![1, 2, 3];
    let show = || {
        println!("list in side the colosure: {:?}", list);
    };

    show();
    println!("list after calling the show: {:?}", list);
```

上面的例子中，show这个闭包不接受任何参数，但capture了上面的list，show被调用后，list所有权没有被show拿走，依然还在，第7行正常打印list。

那么，如果闭包就是像move走所有权呢？需要在闭包的前面加上move：

```rust
    let list = vec![1, 2, 3];
    let show = move || {
        println!("list in side the colosure: {:?}", list);
    };

    show();
    println!("list after calling the show: {:?}", list);
```

此时编译出错：

```rust
error[E0382]: borrow of moved value: `list`
  --> src/main.rs:10:51
   |
4  |     let list = vec![1, 2, 3];
   |         ---- move occurs because `list` has type `Vec<i32>`, which does not implement the `Copy` trait
5  |     let show = move || {
   |                ------- value moved into closure here
6  |         println!("list in side the colosure: {:?}", list);
   |                                                     ---- variable moved due to use in closure
...
10 |     println!("list after calling the show: {:?}", list);
   |                                                   ^^^^ value borrowed here after move
```

因为list的所有权被show拿走了，最后的print对list的访问将会报错。



### mut闭包



### FnOnce和FnMut



## 迭代器
