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

前面讲了Box\<T\> 和 Rc<T\> 两个指针容器，本节要讲的是 RefCell\<T\> ，和 Box\<T\> 一样，都是单所有权属性，且都可以维护可变资源，那么到底 RefCell\<T\> 和  Box\<T\>有什么不一样呢？

<!--more-->

## RefCell\<T\> 和 Box\<T\> 的区别

Rust 的优点（可能也是别人认为的缺点）就是尽可能的在编译时发现问题，令有可能产生问题的代码编译失败，强迫程序员按照既定的规则写代码，从而保证高质量代码。

例如，Rust 有一条规则：引用一个资源，要么是可变且只允许一个可变引用，要么是不可变的但可以多个不可变引用，决不允许同时存在对一个资源进行可变引用和不可变引用：

```rust
#[derive(Debug)]
struct Person {
    age: i32,
    name: String,
}

fn main() {
    let mut a = Person {
        age: 100,
        name: String::from("jack"),
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
    name: String,
}

fn main() {
    let mut a = Box::new(Person {
        age: 100,
        name: String::from("jack"),
    });    
    let b = a.as_ref();
    let mut c = a.as_mut();

    println!("b = {:?}, c = {:?}", b, c);
}
```

上面的第 10 行 b 从Box 那里得到了一个不可变的Person对象引用，这没问题。但 11 行又试图获得一个 可变的 Person 对象引用，这就违反了 Rust 最基本的规则。

那么 RefCell\<T\>  呢？当然也遵守，因为一个可变引用和不可变引用同时存在是很危险的，这会导致数据或者逻辑不一致。但其区别就是，RefCell\<T\> 是运行时期才检查，编译时期是可以通过的，一旦运行时被发现违反了这个规则，程序就会panic：

```rust
fn main() {
    let x = RefCell::new(Person {
        age: 99,
        name: String::from("jack"),
    });

    let mut y = x.borrow_mut();
    y.age = 100;
    println!("x = {:?}, y = {:?}", x, y);

    let z = x.borrow();
    
    println!("x = {:?}, y = {:?}, z = {:?}", x, y, z);
}
```

和 Box\<T\> 和 Rc\<T\> 不一样，RefCell\<T\> 不支持解引用操作（\*a），必须显式调用 borrow_mut 方法才能获得其封装的 RefMut\<T\> 对象， RefMut\<T\> 才支持 Deref trait。出于可读性，这里不直接用 “&\*a.borrow_mut() ” 这样的写法，而是依赖  RefMut\<T\> 的封装，假设 y 就是 &T （实际上是 RefMut\<T\>，但其表现得像 &T一样）。

编译上面的代码，不会像 Box\<T\> 那样出现失败，而是成功编译，但运行到 11行时，会产生 panic，因为 Rust 不允许可变引用和不可变引用同时存在：

```rust
x = RefCell { value: <borrowed> }, y = Person { age: 100, name: "jack" }
thread 'main' panicked at 'already mutably borrowed: BorrowError', src/main.rs:19:15
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

也可以看到，一旦 x 被可变引用了，其 value 状态变为被借用，如果两个不可变引用同时存在，那么一点问题都不会有：

```rust
fn main() {
    let x = RefCell::new(Person {
        age: 99,
        name: String::from("jack"),
    });

    let y = x.borrow();
    println!("x = {:?}, y = {:?}", x, y);

    let z = x.borrow();

    println!("x = {:?}, y = {:?}, z = {:?}", x, y, z);
}
```

以上代码正常编译且运行无误。

由此可见，RefCell和Box 本质区别就在于，前者是运行时期检查引用规则，若有违反，则panic，后者在编译时期检查引用规则，若有违反，则编译失败。



## RefCell\<T\> 配合 Rc\<T\> 实现多引用可变类型

我们可以利用Rc\<T\>的 clone 方法绕过 Rust 的运行时检查，从而获得同时存在多个可变引用的变量：

```rust
fn main() {
    let x = Rc::new(RefCell::new(Person {
        age: 99,
        name: String::from("jack"),
    }));

    let y = Rc::clone(&x);
    y.borrow_mut().age = 100;
    println!("x = {:?}, y = {:?}", x, y);

    let z = Rc::clone(&x);
    z.borrow_mut().age = 200;

    println!("x = {:?}, y = {:?}, z = {:?}", x, y, z);
}
```

可以看到，上面的代码首先用 Rc\<RefCell\<T\>\> 的方式创建了一个对象 x，它终究是Rc，因此，可以利用Rc::clone() 方法又获得一个 y 和 z，然后我们通过点操作直接调用放在 Rc 里面的 Refcell 的 borrow_mut 方法，最终达到了同时多个可变引用的目的。



## RefCell\<T\> 导致的内存泄漏问题

由于我们可以使用RefCell\<Rc\<T\>\> 创建这么一种类型：即可变且多引用的类型，那么，如果有两个节点，相互引用，那么它们的 Rc 永远无法到达0，也就无法被析构，最终造成内存泄漏：

 ```rust
 use std::{rc::Rc, cell::RefCell};
 
 #[derive(Debug)]
 enum Node {
     Next(i32, RefCell<Rc<Node>>),
     Nil,
 }
 
 use Node::Next;
 use Node::Nil;
 
 impl Drop for Node {
     fn drop(&mut self) {
         if let Next(t, _) = self {
             println!("the node will be drop, {}", t);
         }
     }
 }
 
 fn main() {
     let x = Rc::new(Next(1, RefCell::new(Rc::new(Nil))));
     let y = Rc::new(Next(2, RefCell::new(Rc::new(Nil))));
 
     if let Next(_, next) = x.as_ref() {
         *next.borrow_mut() = Rc::clone(&y);
     }
     if let Next(_, next) = y.as_ref() {
         *next.borrow_mut() = Rc::clone(&x);
     }
 
     let p = Rc::new(Next(999, RefCell::new(Rc::new(Nil))));
 }
 ```

上面的代码中，首先 x 和 y 都是 Rc，然后它们内部又有指针指向另一个Rc，初始化的时候先都初始化为 Nil，然后通过RefCell\<T\> 的可变性，把它们都各指向对方，于是，main 函数结束的时候，x 的Rc减 1，但原始值为 2，因为 y 有它的引用，所以 2 - 1 = 1，不为 0，x不析构，同理，y 也不会析构，于是内存泄漏了。

上面的 p 是用于观察正常情况下析构是否进行的。

