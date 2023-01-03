---
title: Rust所有权
date: 2023-01-02 16:17:40
published: false
tags:
categories:
- Rust
---

Rust的所有权应该是Rust语言最难理解的一个，是初学者学习Rust遇到的第一个也是最大的陡坡。很多语言几乎不需要去花时间学习资源管理（Java，Go，Python等，C++当然是除外的），所以那些语言很流行，很受欢迎，因为只需要学习一下语法，关键字就能投入到项目中去了，但Rust学习单单是所有权（相当于资源管理）这一块内容，就可以成功劝退绝大部分程序员。

这是Rust的劣势，决定了Rust流行不起来，也是Rust的优势，决定了Rust效率和正确率上都远优于其它语言（甚至包括C语言）。今天写个总结，把Rust所有权内容说一说。



## 简单的生命周期

从最简单的说起，let表示绑定某个指针到某个对象上，当离开作用域时，对象被销毁，指针进而变为invalid：

```rust
fn main() {
    let s = String::from("hello rust!");
}
```

例如上面的代码，离开main函数，就变成invalid了，String对象也销毁了。

上面的代码相当于这样：

![绑定关系](https://www.jackhuang.cc/svg/rust_所有权.svg)

## 从最简单的所有权关系说起

### 赋值

其实Rust难，就难在下面这个简单代码里面，很多人是无法接受Rust这种规定，但其实Rust这么干也是用心良苦。

```rust
    let s = String::from("hello rust!");
    let t = s;
    println!("t = {}", t);
    println!("s = {}", s);
```

以上代码是不能通过编译的，因为s赋值给t后，s将会变成invalid，也就是悬空指针，这个时候不能对s进行任何访问操作，其对象关系图如下：



### 返回值



## 使用借用



## 借用的限制



## 使用拷贝



## 数组的特殊情况



## 使用RC计数器



