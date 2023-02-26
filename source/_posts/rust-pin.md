---
title: Rust 异步编程（三）：Pin 详解
date: 2023-02-26 16:07:13
tags:
categories:
- Rust
---

本节不讲异步编程。但 Pin 最先出现的地方就是在我们使用 Future 的时候，因此放入 异步编程里面讲了。

Pin 的确是一个很特殊的容器，官方文档和其它查到的资料实在晦涩难懂，即使看中文版我也觉得没有解释太清楚（可能自己笨）。还是那一句话，如果文档和各种资料都看不懂的话，那么，看源码就可以了。

通过对源码的了解，Pin 却又是非常简单的容器，甚至可以这么说，它的特殊之处就是因为它比较简单。Pin 只是实现了这么一个特殊功能的容器：若发现 T 是一个 pinned struct，即禁止内存移动的结构体，则 Pin 禁止返回 &mut T。

下面，我们把官方文档拆开来讲，深入浅出的彻底搞懂 Pin 吧。



<!--more-->

## Rust  的原始指针

在讲 Pin 之前，我们先大概了解一下 Rust 的原始指针，这里会原始到不使用 Rust 的任何函数。比如，我们要获得 String 的指针：

```rust
    let s = String::from("hello rust!");
    let p: *const String = &s;

    unsafe {
        println!("the address of s is {:?}, its content is {}", p, *p);
    }
```

上面的代码第 2 行，就是获得字符串 s 的地址，并让 p 存放这个地址，注意这里用的 *const String，和 C++ 语言一样，表示这个 p 不允许修改 s 的内容，当然可以用 mut 指针：

```rust
    let mut s = String::from("hello rust!");
    let p: *mut String = &mut s;

    unsafe {
        (*p).push_str(" this is greeting to rust");
        println!("the address of s is {:?}, its content is {}", p, *p);
    }
```

由于 *p 是一个危险操作，Rust 需要我们知道这么做是有风险的，于是需要放在 unsafe 块里面。

好，了解了 Rust 的原始指针后，可以看是 Pin 解析之旅了。



## Pin 的设计思想

开篇说了，Pin 的目的就是若 struct 是一个被标记为 pinned 的结构体，则它不让我们获得 &mut T。若你现在不理解这句话的意思，我现在开始从头开始讲，回过头来看这句话那么就完全理解 Pin 了。

首先，Rust 若想移动一块 struct，就需要 &mut T。比如下面这个代码：

```rust
    let mut s = String::from("hello rust!");
    let mut t = String::from("hello jack!");

    std::mem::swap(&mut s, &mut t);

    println!("s = {}, t = {}", s, t);
```

上面的代码会输出：

```rust
s = hello jack!, t = hello rust!
```

即 s 和 t 的内容被交换了，不仅仅是 std::mem::swap 这个函数，只要是内存移动，只要获得 &mut T 就能实现。因此，如果让一个 struct 对象的内存不能移动，那么，就得想办法想获取 &mut T 的时候阻止返回即可，按照 Rust 的哲学，这个阻止动作要发生在编译期。

那么，怎么阻止呢？这就用到 Pin，只要是放在 Pin 内的 对象，且对象被标记为 pinned，那么都无法返回 &mut T。这样就阻止了内存移动。

 

## 内存移动的坑

在讲 Pin 的细节前，解释一下，为什么有时我们要阻止内存移动。其实默认情况下，任何 primitive 类型都是可以移动的，由这些 primitive 类型组成的 struct 也当然可以移动，也就是默认情况下，它们都是 unppinned。什么情况下我们要改成 pinned 呢？最典型的的就是 self-reference 的struct，这个在官方文档由详细的列举，这里给一个简单的代码实例：

```rust
#[derive(Debug)]
struct Node {
    name: String,
    p_name: *const String,
}

impl Node {
   pub fn new(name: String) -> Node {
        let n = Node {
            name, 
            p_name: std::ptr::null() as *const String,
        };
        n
   } 

   pub fn init(&mut self) {
        self.p_name = &self.name;
   }
}

fn test() {
    let mut node1 = Node::new("jack".to_string());
    node1.init();

    let mut node2 = Node::new("rose".to_string());
    node2.init();

    std::mem::swap(&mut node1, &mut node2);

    unsafe {
        println!("node1.name = {}, node1.p_name = {:?}, *node1.p_name = {}",
                node1.name, node1.p_name, *node1.p_name);

        println!("node2.name = {}, node2.p_name = {:?}, *node2.p_name = {}",
                node2.name, node2.p_name, *node2.p_name);
    }
}

fn main() {
    test();
}
```

上面的代码中，有一个小细节要解释一下，就是 Node 的 init 方法为什么不写在 new 方法里面，如果你调试会发现 new 方法是一个静态方法，没有 self 或者 &self，因此里面的对象地址每一次调用的是相同（有可能是出于优化目的，我没有研究），只有绑定到 new 返回的地方的变量上才能拿到 self 或者 &self，因此，此时才能拿到 name 字段的地址。

接着我们可以看到，node1 和 node2 在调用 swap 后会被交换内容，即 name 字段被交换了，实际上地址也被交换了，但也因此 *p_name 还是指向原来的内容，即它们指向的对象的值没有被交换：

```rust
node1.name = rose, node1.p_name = 0x7ffeefbff2f8, *node1.p_name = jack
node2.name = jack, node2.p_name = 0x7ffeefbff318, *node2.p_name = rose
```

可以看到，node1.name 由原来的 “jack” 变成了 “rose”，但对应的 *p_name 还是 “jack”！

Pin 的出现，就是让这种误操作变成编译失败。

但其实细想，这个结果是符合逻辑的，只是不符合在设计这个 struct 的时候的初衷，因为它的设计者认为，p_name 是保存 name 的地址，因此它的内容应该是要和 name 保持一致才对。



## 构造 Pin 容器的基本规则

一个 T 想放入 Pin 中，必须实现 Deref trait，也即能用 * 解引用，因此，能放入 Pin 的，要么是一个生命周期覆盖整个 Pin 对象的 T 对象引用，要么是类似 Box 这种容器。

例如：

```rust
    let mut n1 = Node::new("jack".to_string());
    let mut node1 = Pin::new(&mut n1);
```

上面的代码 n1的生命周期覆盖了 node1 的生命周期，因为 node1 作为 Pin\<&Node\> 引用了 n1。



## Pin 是如何保护我们的 struct 的

那么 Pin 是如何让 swap 这种误操作变成编译错误的呢？针对不同的 T 所在内存 ，分为两种情况。



### 栈数据保护

#### get_mut

get_mut 返回 &mut T，那么如何禁止对于不允许内存移动的 struct 而产生编译错误呢？首先，前面说过，默认情况下 primitive 类型都是 unpinned 的，其组成的 struct 也是 unpinned 的，那么我们必须要有一个 pinned 类型，把这个类型放进 struct 中，于是 struct 就变成了pinned了，这个类型就是 PhantomPinned。于是，我们的例子中，Node 首先增加这个字段，这样 Node 也就是一个 pinned 类型的 T 了：

```rust
#[derive(Debug)]
struct Node {
    name: String,
    p_name: *const String,
    _ph: PhantomPinned,
}
```

这时，我们若调用 get_mut 想试图返回 &mut T 其结果将会报错：

```rust
fn test() {
    let mut n1 = Node::new("jack".to_string());
    node1.init();
    let mut node1 = unsafe {Pin::new_unchecked(&mut n1)};

    let mut n2 = Node::new("rose".to_string());
    node2.init();
    let mut node2 = unsafe {Pin::new_unchecked(&mut n2)};

    std::mem::swap(node1.get_mut(), node2.get_mut());

    unsafe {
        println!("node1.name = {}, node1.p_name = {:?}, *node1.p_name = {}",
                node1.name, node1.p_name, *node1.p_name);

        println!("node2.name = {}, node2.p_name = {:?}, *node2.p_name = {}",
                node2.name, node2.p_name, *node2.p_name);
    }
}
```

上面的代码中，第 10 行会收到编译失败，其中会提示：

```rust
error[E0277]: `PhantomPinned` cannot be unpinned
   --> src/main.rs:34:20
    |
34  |     std::mem::swap(node1.get_mut(), node2.get_mut());
    |                    ^^^^^ ------- required by a bound introduced by this call
    |                    |
    |                    within `Node`, the trait `Unpin` is not implemented for `PhantomPinned`
```

这个错误是怎么实现的呢？这时因为 Pin 的 get_mut 方法，需要 T 实现 unpinned 这个 trait，但由于 Node 增加了 PhantomPinned 字段，导致不满足这个条件，于是报错：

```rust
    pub const fn get_mut(self) -> &'a mut T
    where
        T: Unpin,
    {
        self.pointer
    }
```

可以看到，where 中对 T 进行了 Unpin 限制，由于新增字段 PhantomPinned 我们的 Node 是不满足的。

另外，还有一个地方要注意，我们在初始化 Pin 对象的时候，调用的方法是 new_unchecked，而不是 new，这时因为 Pin 的 new 会检查 T 是否是 unpinned 的，若不是，则编译失败，直接在入口处防止了 pinned 对象的进入，后续也就规避了 swap 内存移动的坑。这里调用 new_unchecked 是相当于把这个 trait bound 检查后置了。



#### as_mut

如果把 get_mut 换成 as_mut 呢？Pin 的as_mut 方法代码如下：

```rust
    #[stable(feature = "pin", since = "1.33.0")]
    #[inline(always)]
    pub fn as_mut(&mut self) -> Pin<&mut P::Target> {
        // SAFETY: see documentation on this function
        unsafe { Pin::new_unchecked(&mut *self.pointer) }
    }
```

可以看到，返回的是 Pin 对象而不是我们期待的 &mut T，对比 Box的 as_mut 方法：

```rust
pub trait AsMut<T: ?Sized> {
    /// Converts this type into a mutable reference of the (usually inferred) input type.
    #[stable(feature = "rust1", since = "1.0.0")]
    fn as_mut(&mut self) -> &mut T;
}
```

其返回的却是 swap 要的 &mut T，这就是最本质的区别。因此，Pin 可以阻止内存移动，而 Box 不行。因此，换成 as_mut 方法后，会产生如下编译错误：

```rust
note: expected `&mut _`, found struct `Pin`
   --> src/main.rs:34:20
    |
34  |     std::mem::swap(node1.as_mut(), node2.as_mut());
    |                    ^^^^^^^^^^^^^^
    = note: expected mutable reference `&mut _`
                          found struct `Pin<&mut Node>`
```

即入参不符合期望。如果细心看编译错误，还会提示我们应该改成下面的代码：

```rust
    |
49  |     std::mem::swap(&mut node1.as_mut(), node2.as_mut());
    |                    ~~~~~~~~~~~~~~~~~~~
help: consider mutably borrowing here
    |
49  |     std::mem::swap(node1.as_mut(), &mut node2.as_mut());
```

如果我们按照提示的方法修改，也即我们取返回值的 &mut 引用，也即我们把 &mut Pin\<&T\> 给了 swap 函数，那么会发现内存并没有交换：

```rust
    std::mem::swap(&mut node1.as_mut(), &mut node2.as_mut());
```

其输出为：

```rust
node1.name = jack, node1.p_name = 0x7ffeefbff318, *node1.p_name = jack
node2.name = rose, node2.p_name = 0x7ffeefbff2f8, *node2.p_name = rose
```

可以看到，内存没有变动，而且符合我们设计 Node 时的初衷，即 name 和 *p_name 内容一致。这是因为 Pin 返回的是一个新对象，老对象放在里面丝毫未动。Pin 虽然无法阻止我们在外面写出 &mut T 这样的代码，但它通过返回一个新对象保护了 T 的内存被移动。



### 堆数据保护

#### get_mut

如果把 T 放入堆里面呢？比如用 Box 容器，此时，我们的 get_mut 讲无法使用，因为 get_mut 是放在这个 Pin 特化下面的：

```rust
impl<'a, T: ?Sized> Pin<&'a mut T> 
```

上面的这一行代码， ?Sized 表示 T 必须是一个在编译时期未知具体大小的类型，比如引用，但 Box 是一个编译时期已知大小的类型（它的成员是指针，固定大小），因此，Pin\<Box\<T\>\> 无法调用 get_mut。



#### as_mut

Pin\<Box\<T\>\> 中 Box\<T\> 实际就是一个栈上的数据（ T 在堆上），因此此时 as_mut 的情况和之前讨论在站上的数据保护一样，由于 as_mut 返回的是一个新的 Pin\<Box\<T\>\> 对象，因此，无法通过编译，即使通过引用符号转为 &mut T 类型，虽然可以通过编译并运行，但因为是新对象而不会造成原来的 T 对象有内存移动。这种情况完全和栈数据时的情况一样。



## 前方高能：直接将 Pin 的引用传给 swap

前面说的都是 Pin 如何通过 trait bound 或者新创建 Pin 对象的方法防止我们通过 get_mut 和 as_mut 这两个方法获得 &mut T，从而使得 swap 函数编译报错阻止了内存移动的坑。但如果我们直接把 Pin 当作一个对象，直接把这个对象的 &mut 引用传入 swap 呢？

### 栈的情况

```rust
fn test() {
    let mut n1 = Node::new("jack".to_string());
    n1.init();
    let mut node1 = unsafe {Pin::new_unchecked(&mut n1)};

    let mut n2 = Node::new("rose".to_string());
    n2.init();
    let mut node2 = unsafe {Pin::new_unchecked(&mut n2)};

    std::mem::swap(&mut node1, &mut node2);

    unsafe {
        println!("node1.name = {}, node1.p_name = {:?}, *node1.p_name = {}",
                node1.name, node1.p_name, *node1.p_name);

        println!("node2.name = {}, node2.p_name = {:?}, *node2.p_name = {}",
                node2.name, node2.p_name, *node2.p_name);
    }
}
```

可以看到，如此一来，即使我们的 Node 有 PhantomPinned 字段，变成了 pinned 对象，但我们通过直接把 Pin 对象引用传给了 swap 函数绕过了 Pin 为我们做的检查，编译还是可以顺利通过，运行符合我们的预期：

```rust
node1.name = rose, node1.p_name = 0x7ffeefbff2f8, *node1.p_name = rose
node2.name = jack, node2.p_name = 0x7ffeefbff318, *node2.p_name = jack
```

这时因为 Pin\<T\> 被当作一个整体被 swap 了，也就是 swap 的是 Pin 对象而不是 T 对象， Pin 对象没有self-reference 问题，因此这么做结果是没问题的。



### 堆的情况

轮到堆的情况：

```rust
fn test() {
    let mut n1 = Node::new("jack".to_string());
    let mut node1 = Box::pin(&mut n1);
    node1.init();

    let mut n2 = Node::new("rose".to_string());
    let mut node2 = Box::pin(&mut n2);
    node2.init();

    std::mem::swap(&mut node1, &mut node2);

    unsafe {
        println!("node1.name = {}, node1.p_name = {:?}, *node1.p_name = {}",
                node1.name, node1.p_name, *node1.p_name);

        println!("node2.name = {}, node2.p_name = {:?}, *node2.p_name = {}",
                node2.name, node2.p_name, *node2.p_name);
    }
}
```

 这回我们也是把 Pin 对象当作 T 然后转为 &mut T 传给 swap 函数了。此时结果当然也是令我们满意的：

```rust
node1.name = rose, node1.p_name = 0x7ffeefbff2e8, *node1.p_name = rose
node2.name = jack, node2.p_name = 0x7ffeefbff308, *node2.p_name = jack
```

不管是 name 字段还是 p_name 字段，都交换了，之前提到的内存移动的坑没有了。这是因为 Pin\<T\> 中 T 没有 self-reference 问题，它被整体交换了，于是结果符合预期。



## 为什么 Future 要放在 Pin\<Box\<T\>\> 中

这是因为，Future 对象相当于是一个函数转过去的对象，如果我们在函数中定义了一个数组，然后调用一个 IO 阻塞函数，函数中引用这个数组并往里填数据，那么这之中将会触发一次 await，也即 Future 对象会在 IO 阻塞完成的时候重新 send 到队列中给 executor 执行，这就是一个内存移动过程：

```rust
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

例如上面官方的例子，第四行会有 IO 阻塞，此时 poll（回忆上一节讲的，await 相当于调用了 poll 方法）返回 Pending，executor 收到返回值 Pending转去执行别的 Future 了，当 read_into_buf_fut 完成后，会重新被 send 进队列等待 executor 再次 poll 后继续执行第 5 行代码，此时上面的代码中，x 和第三行的 &x 就相当于 self-reference，若没有 Pin，将会产生内存移动的坑。有了 Pin，内存移动就会像上一小节那样符合预期的进行移动。



## 总结

1、Pin\<T\> 通过 trait bound 检查（new 和 get_mut 方法）或者返回新的 Pin 对象（as_mut 方法）来防止内存移动的坑；对于 pinned struct 结构产生编译报错，防止我们直接对 T 进行内存移动操作。

2、由于 Pin 的封装，我们把 Pin\<T\> 看成一个整体移动内存的话，则符合预期；

3、由于Pin 一方面阻止了 T 的内存移动bug，另一方面保证了 Pin\<T\> 可以符合预期的进行内存移动，因此，若出现 self-reference 的情况，我们应该用 Pin 封装一层，防止踩坑。

从上面的讨论可以看出， 尽管它的名字叫 Pin，Pin 并不是用来禁止内存移动的，而是保证内存移动能正常进行的类型。

