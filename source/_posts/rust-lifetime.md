---
title: Rust的生命周期（lifetime）
date: 2023-01-19 16:19:54
toc: true
published: false
tags:
- Rust
categories:
- Rust
---

生命周期，即lifetime是Rust最独有的一个特性，早期并没有这个特性，但后来为了辅助Rust的编译器检查生命周期是否合法，也为了调用方方便确认函数或者方法对生命周期的要求就加上去了，也许在未来，这个特性会被优化掉，谁知道呢。做为学习者，我们还是要把这些细节知识补充一下的。

<!--more-->

## 生命周期是什么

生命周期（lifetime）是annotation，即一个标记。仅此而已。不管我们怎么做，生命周期就是一个标记，它没有改变任何东西，尤其不会改变对象的生命周期，尽管我们把它叫做生命周期。所以，遇到生命周期时，记住，它只是一个标记，相当于给编译器看的注释，用于给它提示对象的生命周期的。



## 函数的生命周期

为什么我们要关注生命周期呢，看看下面这个例子，找出一句话的第一个单词：

```rust
fn find_first_word(s: &str) -> &str {
    let words: Vec<&str> = s.split(&[' ', ',', '.']).collect();
    if words.len() > 0 {
        return words.get(0).unwrap();
    }
    s
}
```

可以看到这个





## struct的生命周期



## Rust编译器检查生命周期的三个规则
