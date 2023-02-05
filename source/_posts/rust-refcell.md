---
title: Rust的RefCell容器
date: 2023-01-29 21:43:59
tags:
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

由此可见，RefCell和Box 本质区别就在于，虽然只能有唯一的所有权方，但前者是运行时期检查引用规则，若有违反，则panic，后者在编译时期检查引用规则，若有违反，则编译失败。

当然方法的区别也是有的Box 两个获取 T 引用的方法是 as_ref 和as_mut，而 RefCell 的两个方法是 borrow 和 borrow_mut 。



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

上面的 p 是用于观察正常情况下析构是否进行的。上面的代码输出为：

```rust
the node will be drop, 999
```

x 和 y 不会被析构。

解决方案是引入 weak\<T\> 。weak\<T\> 即若引用，相对于强引用，若引用首先并不获得所有权，也就是若引用计数是不是 0 并不影响对象的析构。因此，weak\<T\> 可能持有一个无效的 T 引用，所以 weak\<T\> 获取 T 引用的时候是返回的 Option\<T\> ，如果 weak\<T\> 已经无效，则返回 None，如果 weak\<T\> 依然有效，则返回Some(T) 。

```rust
n main() {
    let x = Rc::new(Next(1, RefCell::new(Weak::new())));
    let y = Rc::new(Next(2, RefCell::new(Weak::new())));
  
    if let Next(_, next) = t.as_ref() {
        if let None = next.borrow().upgrade() {
            println!("it is None initially");
        }
    }

    if let Next(_, next) = x.as_ref() {
        *next.borrow_mut() = Rc::downgrade(&y);
        println!("the weak count of y is {} and strong count of y is {}", Rc::weak_count(&y), Rc::strong_count(&y));

        if let Some(other) = next.borrow().upgrade() {
            println!("other = {:?}", other);
        }
    }
    if let Next(_, next) = y.as_ref() {
        *next.borrow_mut() = Rc::downgrade(&x);
        println!("the weak count of x is {} and strong count of x is {}", Rc::weak_count(&x), Rc::strong_count(&x));
    }

    let p = Rc::new(Next(999, RefCell::new(Weak::new())));
}
```

上面的代码中，首先第二三行的代码 RefCell::new(Rc::new(Nil)) 被修改为 RefCell::new(Weak::new()) ，可以看到我们直接用了weak::new() 初始化 weak\<T\> 指针，因为 weak\<T\> 并不需要主动初始化，它是 Rc 的弱引用，先有 Rc 才能有 weak\<T\>。

先略过 5 到 9 行，x 和 y 初始化结束后，我们把 x 和 y 互相指向对方，注意第 12 行没有再使用 Rc::clone，而是 Rc::downgrade，与 Rc::clone 返回 Rc 不同，Rc::downgrade 返回一个 weak\<T\> 对象（这也是获得一个可能非None的 weak\<T\> 对象的方法），赋值给了 x 的 weak\<T\>。同理，y 这边也是这样。

这样 x 和 y 都相互引用了对方，但由于是弱引用，因此，即是它们相互引用，也不妨碍最后离开 main 函数时被析构：

```rust
it is None initially
the weak count of y is 1 and strong count of y is 1
other = Next(2, RefCell { value: (Weak) })
the weak count of x is 1 and strong count of x is 1
the node will be drop, 999
the node will be drop, 2
the node will be drop, 1
```

可以看到，最后 x 和 y 都被析构了，我们也打印了weak count 和 strong count，都是 1， 结束的时候，x 和 y 都是Rc，因此 strong count 减 1，x 和 y 析构。weak count 不管多少，都不影响 Rc 被析构。 

weak\<T\> 不拥有所有权，依靠调用 upgrade 来获取 T 引用，upgrade 返回 Option<T\> ，若原 Rc 已经被析构，则返回 None，否则返回 Some(Rc\<T\>)。没有初始化的情况下，则直接返回None。



## 总结

|                | Box\<T\>                                                     | Rc\<T\>                                                      | RefCell\<T\>                                                 | RefMut\<T\>/Ref\<T\>                                         | Weak\<T\>                                                    |
| :------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 所有权规则     | 只有一个所有权                                               | 依靠强引用计数实现多个所有权                                 | 只有一个所有权                                               | 只有一个所有权                                               | 依靠弱引用计数实现多个所有权，但实际并不获得所有权，依靠Option\<T\> 来判断资源是否有效 |
| 什么时候使用   | 1、大资源，想减少拷贝的时候；2、实现多态；3、实现栈分配；4、实现无限大小的对象。 | 需要多个所有权方时。                                         | 若Box\<T\>编译时期规则（一个资源同一个时间内，要么只有一个可变引用，要么有多个不可变引用，禁止可变引用和不可变引用同时存在）检查无法满足需求，需要在运行时检查，则可以使用RefCell\<T\>。 | 由RefCell\<T\>的borrow_mut/borrow方法返回，一般不直接使用。一般情况下，用户不感知这个类的存在。可以认为borrow_mut/borrow返回的就是对应的资源引用即可。 | 由 Rc\<T\>的downgrade方法返回，相比Rc\<T\>的强引用，Weak\<T\>用于弱引用。 |
| 解引用方法     | 实现Deref trait 和 Copy trait，即let b = \*a。               | 因为有引用计数规则，不能直接拿走所有权，因此不能直接解引用获取资源，即禁止let b = \*a，必须通过 Rc::clone 和self.as_ref 方法对资源进行访问。当然写成 let b = &\*a（这样没有拿走所有权，不违反规则） 也可以，但可读性差。 | 同样不允许let b = \*a这样的操作。                            | 同样不允许let b = \*a这样的操作。                            | 同样不允许let b = \*a这样的操作。应该使用self.upgrade方法（返回Option）判断资源是否有效。 |
| 不可变引用方法 | as_ref                                                       | as_ref                                                       | borrow                                                       | 直接点运算操作即可                                           | upgrade判断返回值Option                                      |
| 可变引用方法   | as_mut                                                       | 不支持，但可以结合 RefCell\<T\> 实现，即Rc\<RefCell\<T\>\>   | borrow_mut                                                   | 直接点运算操作即可                                           | 不支持，但可以结合Rc\<RefCell\<T\>\>实现。                   |

注意，以上所有的容器都无法在多线程的时候使用。

那么，多线程怎么办呢？下面就开始无惧并发的学习。
