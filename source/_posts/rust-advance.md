---
title: Rust 的高级特性
date: 2023-02-03 18:57:38
published: false
tags:
- Rust
- Rust advanced fetures
categories:
- Rust
---

今天讲一下 Rust 的高级特性。

包括：

1、unsafe 代码；

2、高级trait；

3、高级类型；

4、高级函数和闭包；

5、宏。

<!--more-->

## unsafe 代码

### 使用 raw 指针

最 unsafe 的行为莫过于直接使用 raw 指针，这时会绕过 Rust 的安全检查，比如经典的不能同时有可变和不可变引用规则，若使用 raw 指针，那么就需要我们很清楚的说，这是 unsafe 代码，Rust 才放心的运行：

```rust
   let mut a = 5;

    let x = &a as *const i32;
    let y = &mut a as *mut i32;

    unsafe {
        println!("*x = {}, *y = {}", *x, *y);
    }
```

类似的还有比如存在两个&mut 引用同一个变量：

```rust
fn split_at_mut(a: &mut [i32], index: usize) -> (&mut [i32], &mut [i32]) {
    assert!(a.len() >= index);
    (&mut a[..index], &mut a[index..])
}

fn main() {
    let mut a = [1, 2, 3, 4, 5];
    println!("split at 3: {:?}", split_at_mut(&mut a, 3));
}
```

上面的代码中，split_at_mut 函数最后一行两次 &mut a 会违反引用规则：

```rust
2 | fn split_at_mut(a: &mut [i32], index: usize) -> (&mut [i32], &mut [i32]) {
  |                    - let's call the lifetime of this reference `'1`
3 |     assert!(a.len() >= index);
4 |     (&mut a[..index], &mut a[index..])
  |     -----------------------^----------
  |     |     |                |
  |     |     |                second mutable borrow occurs here
  |     |     first mutable borrow occurs here
  |     returning this value requires that `*a` is borrowed for `'1`
```

这是因为 Rust 担心两个引用数组所包含的区间会相互覆盖。如果我们告诉 Rust 这个是 unsafe 代码，那么， Rust就会让编译通过并运行：

```rust
fn split_at_mut(a: &mut [i32], index: usize) -> (&mut [i32], &mut [i32]) {
    assert!(a.len() >= index);
    let p = a.as_mut_ptr();
    unsafe {
        (slice::from_raw_parts_mut(p, index),
         slice::from_raw_parts_mut(p.add(index),  a.len() - index))
    }
}
```



### 调用 unsafe 函数或者方法

如果一个函数或者方法是 unsafe 的，那么当然我们也必须告诉 Rust 我们知晓这个行为 unsafe，运行吧。



### 调用 C 函数也是 unsafe 的

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```

上面的代码中，调用 C 函数的 abs，extern "C" 表示调用 C 函数库。

当然反过来， C 函数调用 Rust 函数却是 safe 的，这里不展开来说。关于 C 和 Rust 互调以后有机会再说。



### 访问静态可变变量也是 unsafe 的

Rust 中，如果按照存储特性来分，有三种类型：

const：运行时存放于常量区，非运行时被固化在文件中；const 数据允许出现在任何地方定义。

static：进程开始后有固定的内存地址；static 数据允许出现在任何地方定义。

堆栈：堆栈数据则动态分配。堆栈数据则只能在局部中定义。

这里，若 static 是可变的，那意味着很多地方都可以去访问 static mut，这当然相当于打破了一个变量值允许一个 mut 引用的规则：

```rust
fn main() {
    static mut a: i32 = 9;
    a += 1;
}
```

  此时编译输出：

```rust
 --> src/main.rs:5:5
  |
5 |     a += 1;
  |     ^^^^^^ use of mutable static
```

明确告诉我们，去使用一个 static mut 的变量是不允许的，应该用unsafe：

```rust

fn main() {
    static mut a: i32 = 9;

    unsafe {
        a += 1;
    }
}
```



### 实现 unsafe trait 也当然是 unsafe 代码

 这个调用了 unsafe 函数或者方法一样。



### 使用union 也是 unsafe 的

和 C 一样，Rust 也有union，它是 unsafe的。



### 什么时候用 unsafe

自己能保证没问题的时候，例如上面的 split 数组，很明显，不管怎么样，split 出来的数组都不会相互有空间的覆盖，虽然 Rust 有这方面的考虑，但既然我们能确定不会有，那么我们就告诉 Rust，这个是已知 unsafe 代码，已确保无误。
