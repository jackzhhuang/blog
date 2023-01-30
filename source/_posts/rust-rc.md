---
title: Rust的Rc容器
date: 2023-01-29 10:14:33
tags:
- Rust
- Smart Pointer
categories:
- Rust
---

前面讲了 Box\<T\> ，和一般的变量一样，它只能有一个所有权方，如果需要有多个所有权，即实现类似（不完全一样，因为Rust只要你不说，都是不可变的）python那样的所有权复制，就需要Rc\<T\>了。

<!--more-->

## 使用Rc\<T\> 实现多个不可变资源共享

之前的Box\<T\>实现了一个如同以下的链表：

![Box\<T\>实现无限链表](https://www.jackhuang.cc/svg/conforbox.svg)

其代码如下：

```rust
#[derive(Debug)]
enum List {
    Con(i32, Box<List>),
    Nil,
}

use List::Con;
use List::Nil;

fn main() {
    let a = Con(1, Box::new(Con(2, Box::new(Con(3, Box::new(Nil))))));
    println!("a = {:?}", a);
}
```

如果串入另一个链表如下，Box\<T\>就不能实现了，因为Box\<T\>是只能有一个所有权方的：

![两个链表串起来](https://www.jackhuang.cc/svg/conforrc.svg)

写成代码如下：

```rust
#[derive(Debug)]
enum List {
    Con(i32, Box<List>),
    Nil,
}

use List::Con;
use List::Nil;

fn main() {
    let common_list = Box::new(Con(2, Box::new(Con(3, Box::new(Nil))))); 
    let a = Con(1, common_list);
    let b = Con(101, common_list);

    println!("a = {:?}", a);
    println!("b = {:?}", b);
}
```

编译错误信息如下：

```rust
   Compiling greeting v0.1.0 (/Users/jack/Documents/code/rust/vsrust/greeting)
error[E0382]: use of moved value: `common_list`
  --> src/main.rs:14:22
   |
12 |     let common_list = Box::new(Con(2, Box::new(Con(3, Box::new(Nil))))); 
   |         ----------- move occurs because `common_list` has type `Box<List>`, which does not implement the `Copy` trait
13 |     let a = Con(1, common_list);
   |                    ----------- value moved here
14 |     let b = Con(101, common_list);
   |                      ^^^^^^^^^^^ value used here after move
```

可见，common_list在第12行的时候就给了 a 链表，b 链表想用是用不了了。此时就需要Rc\<T\>，它和Box\<T\>不同之处就是它实现了多个不可变对象的共享资源（Box\<T\>可变不可变资源都可以，但只能有一个所有权方）：

```rust
use std::rc::Rc;

#[derive(Debug)]
enum List {
    Con(i32, Rc<List>),
    Nil,
}

use List::Con;
use List::Nil;

fn main() {
    let common_list = Rc::new(Con(2, Rc::new(Con(3, Rc::new(Nil))))); 
    let a = Con(1, Rc::clone(&common_list));
    let b = Con(101, Rc::clone(&common_list));

    println!("a = {:?}", a);
    println!("b = {:?}", b);
}
```

此时可以正常编译并运行。

注意，Rc\<T\> 对象之间在赋值的时候需要使用 Rc::clone 方法，如果直接使用赋值，还是会走到Rust的默认行为：所有权转移。



## 顺带一提：clone和copy方法

顺带提一下 clone 和 copy 方法。clone 和 copy 都是带有复制的意思，但 copy 则比较暴力，是资源的安位拷贝，而 clone 则是和具体是类型实现相关，比如上面的 Rc::clone 实现的是浅拷贝，并且引用计数加 1。而String 的 clone 则是字符串拷贝：

```rust
fn main() {
    let b;
    {
        let a = String::from("hello world!");
        b = a.clone();
        std::mem::drop(&a);
    }
    println!("b = {:?}", b);
}
```

上面的代码中，a 虽然超出作用域，甚至调用了drop，但 b 依然能正常打印，因为 String的 clone 是深拷贝。



## 打印计数

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("hello world!"));
    println!("count = {}", Rc::strong_count(&a));
    let b = Rc::clone(&a);
    println!("count = {}", Rc::strong_count(&a));
    {
        let c = Rc::clone(&a);
        println!("count = {}", Rc::strong_count(&a));
    }
    println!("count = {}", Rc::strong_count(&a));
    let d = Rc::clone(&a);
    println!("count = {}", Rc::strong_count(&a));
}
```

上面的代码中，每次赋值或者走出作用域后都打印一次Rc的内部计数：

```rust
count = 1
count = 2
count = 3
count = 2
count = 3
```

可见，每次clone，引用计数加 1，对象被销毁，则计数减 1，为 0 时，资源被析构。

为什么这里用的是 Rc::strong_count 呢？为什么不是调用 Rc::count，因为还有一个 Rc::weak_count 与 Rc::strong_count 对应。这里就引出了一个问题：资源的相互引用。这在引入RefCell\<T\>  的时候将可能会发生，留待下节讲。



## Rc\<T\> 的方法

### *解引用

和Box不一样，Box 我们可以通过解引用的方式获得 T 的所有权，但如果我们对Rc进行解引用，那么会编译错误，因为这样越过 Rc 的计数法则：

```rust
#[derive(Debug)]
struct Person {
    age: i32,
}

fn print_age(p: &Person) {
    println!("the age of p is {}", p.age);
}

fn main() {
    let x = Rc::new(Person {
        age:99,
    });
    let y = *x;
    println!("x = {:?}, y = {:?}", x, y);
}
```

我们只能得到 Rc\<T\>  的引用：

```rust
    let x = Rc::new(Person {
        age:99,
    });
    let y = &*x;
    println!("x = {:?}, y = {:?}", x, y);
```

修改成 &*x 后 y 将是 Person 的对象引用，一切正常。

这种方式也是安全的，因为编译器在编译的时候就会检查无误，不会出现悬挂指针。



### as_ref

调用 &*x 可读性实在是比较差，调用 as_ref 则清晰很多：

```rust
    let x = Rc::new(Person {
        age:99,
    });
    let y = x.as_ref();
    println!("x = {:?}, y = {:?}", x, y);
```

和 &*x 等效。



### 没有 as_mut

Rc\<T\> 不会提供可变 T 引用方法，因为它是针对不可变 T 来设计的。



## 什么时候使用 Rc\<T\> 

如果 Box这种只能有一方所有权的容器无法满足需求，且不涉及可变资源，那么就可以使用 Rc\<T\> 了。简单来说可以按以下步骤来确定是否可以使用Rc\<T\>：

1、能用 Box\<T\> 那就用；

2、若不能用  Box\<T\>，比如需要多个所有权方，且不涉及可变资源，那么使用 Rc\<T\>。



## 总结

1、Rc\<T\> 和 Box\<T\> 一样，都实现了Deref 和 Drop 这两个 trait，即解引用（*a  操作）和 自动析构；

2、Rc\<T\> 通过引用计数维护不可变资源是否应该释放，当引用计数为 0 的时候资源将被释放；

3、应该通过 Rc::clone 来赋值，因为不同的类型 clone 有不同的实现，而如果不用 clone 方法则会发生所有权转移；

4、和Box\<T\> 一样，Rc\<T\> 也是一个单线程容器，不允许多线程使用；

5、使用as_ref 提高可读性。
