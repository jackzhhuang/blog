---
title: Rust的RefCell容器
date: 2023-01-29 21:43:59
published: false
tags:
- Rust
- RefCell
categories:
- Rust
---

前面讲了Box\<T\> 和 Rc<T\> 两个指针容器，简单总结：Box\<T\> 只能有一个所有权方，而 Rc<T\> 依靠引用计数可以有多个，两个都是维护不可变资源的，且只能单线程使用。还有注意 Rc<T\> clone方法才能做到正确的带引用计数浅拷贝。

那么，如果我们想有一个可变资源指针容器呢？那就需要RefCell\<T\>  了。

<!--more-->
