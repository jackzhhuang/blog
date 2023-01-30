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

前面讲了Box\<T\> 和 Rc<T\> 两个指针容器，简单总结：Box\<T\> 只能有一个所有权方，而 Rc<T\> 依靠引用计数可以有多个，Box\<T\>可以维护可变不可变资源，而 Rc<T\>  只能是不可变资源，相同点是都只能单线程使用。最后还要注意 Rc<T\> clone方法才能做到正确的带引用计数浅拷贝。

本节要讲的是 RefCell\<T\> ，和 Box\<T\> 一样，都是单所有权属性，且都可以维护可变资源，那么到底 RefCell\<T\> 和  Box\<T\>有什么不一样呢？

<!--more-->

## RefCell\<T\> 和 Box\<T\> 的区别

Rust 的优点（可能也是别人认为的缺点）就是尽可能的在编译时发现问题，令有可能产生问题的代码编译失败，强迫程序员按照既定的规则写代码，从而保证高质量代码。

例如，Rust 有一条规则：引用一个资源，要么是可变且只允许一个可变引用，要么是不可变的但可以多个不可变引用，决不允许同时存在对一个资源进行可变引用和不可变引用：

```rust
#[derive(Debug)]
struct Person {
    age: i32,
}

fn main() {
    let mut a = Person {
        age: 100,
    };
    let b = &a;
    let mut c = &mut a;

    println!("b = {:?}, c = {:?}", b, c);
}
```

编译会产生失败：

```rust
   |
10 |     let b = &a;
   |             -- immutable borrow occurs here
11 |     let mut c = &mut a;
   |                 ^^^^^^ mutable borrow occurs here
12 |
13 |     println!("b = {:?}, c = {:?}", b, c);
   |                                    - immutable borrow later used here
```

因为此时 b 是不可变引用，而 c 是可变引用，违反了 Rust 的编译规则。

Box\<T\> 当然也遵循这点，编译时期 Rust 也检查：

```rust
#[derive(Debug)]
struct Person {
    age: i32,
}

fn main() {
    let mut a = Box::new(Person {
        age: 100,
    });    
    let b = &*a;
    let mut c = &mut *a;

    println!("b = {:?}, c = {:?}", b, c);
}
```

上面的第 10 行 b 从Box 那里得到了一个不可变的Person对象引用，这没问题。但 11 行又试图获得一个 可变的 Person 对象引用，这就违反了 Rust 最基本的规则。

那么 RefCell\<T\>  呢？当然也遵守，因为一个可变引用和不可变引用同时存在是很危险的，这会导致数据或者逻辑不一致。但其区别就是，RefCell\<T\> 是运行时期才检查，编译时期是可以通过的，一旦运行时被发现违反了这个规则，程序就会panic：

```rust
#[derive(Debug)]
struct Person {
    age: i32,
}

fn main() {
    let mut a = RefCell::new(Person {
        age: 100,
    });    
    let b = &*a.borrow_mut();
    let c = &mut *a.borrow_mut();

    println!("b = {:?}, c = {:?}", b, c);
}
```

和 Box\<T\> 和 Rc\<T\> 不一样，RefCell\<T\> 不支持解引用操作（*a），必须显式调用 borrow_mut 方法才能获得其封装的 RefMut\<T\> 对象， RefMut\<T\> 才支持 Deref trait。所以，此时解引用获得 Person 对象再引用即可获得 Person对象的引用。

因此，b 和 c 都是对 Person 对象的引用，但区别是 b 是不可变引用，而 c 是可变引用。

此时编译成功，因为 RefCell\<T\> 允许它们这么干，但运行时将会panic：

```rust
thread 'main' panicked at 'already borrowed: BorrowMutError', src/main.rs:13:21
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

因此，RefCell\<T\> 和 Box\<T\> 区别即是：前者是运行时检查是否违反前述的引用规定，后者是编译时检查。
