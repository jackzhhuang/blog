---
title: Rust的容器
date: 2023-01-12 22:42:05
published: false
tags:
categories:
- Rust
---

今天熟悉一下Rust的常见容器和其用法。当然本节介绍的容器只是Rust众多数据结构中的一角，只能算是起个头，未来使用容器时，都应该多翻翻文档，并且每次都应该想想：

1、用来做什么？（是数组还是字符串，是需要哈希还是需要有序）

2、对数据访问有什么特别的要求？（顺序访问还是随机访问，要经常插入删除还是多是遍历）

3、是否线程安全？

4、其它各种先决条件。

总之，选择容器一定要看上下文的需求，选择最合适的容器去解决问题。同时，多看文档，多翻文档。

现在我们就从最常用的容器开始吧。

<!--more-->

<!-- toc -->



## 数组Vec

数组Vec是最常用的容器，它非常类似C++的std::vector，底层数据结构也是一块连续的内存，内存不足时，会扩大，并把老数据拷贝到新内存上，因此，非常适合顺序访问和索引访问，非常不适合有删除和插入的场景，因为此时暗含着内存移动的低效操作。

### 高效初始化

```rust
fn main() {
    let array1 = vec![192, 31, 231, -12, 3];

    let mut array2 = Vec::new();
    array2.push("hello world".to_string());
    array2.push("hello rust".to_string());
    array2.push("hello jack".to_string());

    println!("{:?}", array1);
    println!("{:?}", array2);
}
```

上面用了最常用的初始化方法，array1是vec!宏来初始化，array2是直接用Vec::new()来初始化，可以看到我们在定义array2的时候并没有去告诉array2将会放上面类型，直到第6行push第一个String对象的时候才告诉编译器，我们要放的对象是String。Vec只能是单一对象。第6行决定了array2必须放String对象。

前面说了，Vec是会因为自身分配的内存空间不足时，增长内存的，因此会有数据移动的低效操作，array2的初始化就是这样，如果我们能事先告诉array2要存放什么类型，已经即将存放的数量，那么Vec就可以事先分配好空间，保证我们后续放入数据时不会有数据移动操作：

```rust
    let mut array2 : Vec<String> = Vec::new();
    array2.reserve(3);
    array2.push("hello world".to_string());
    array2.push("hello rust".to_string());
    array2.push("hello jack".to_string());
```

上面的reserver方法就是告诉array2实现分配至少3个String对象空间，好让后续的3个push方法调用不产生数据移动操作。

如果事先知道需要分配数组的元素数量，且它们的值都是同一个，那么，更高效的方法是：

```rust
    let array3 = vec![100; 5];
    println!("array3 = {:?}", array3);
```

上面的[100; 5]就表示，分配长度为5个元素，且每个值都是100的数组，这就比前面使用reserver更高效。

还有with_capacity()方法，可以把Vec::new()和reserve()方法都合成一个，例如array2的初始化可以改成：

```rust
    let mut array2 : Vec<String> = Vec::with_capacity(3);
    array2.push("hello world".to_string());
    array2.push("hello rust".to_string());
    array2.push("hello jack".to_string());
```

总之，如果我们实现就能知道有多少元素即将push进Vec里面的话，我们就尽量的把内存分配好，避免数据移动操作。



### 顺序访问

Vec是最适合顺序访问的了，我们可以一个个枚举Vec里面的值：

```rust
    for s in &array2 {
        println!("{}", s);
    }
```

注意，访问array2时我们用了引用，这样访问里面的String对象时，也都变成了引用，这个规则struct也是这样。因为用了引用，后续array2还是有效的指针。



### 索引访问

索引访问也是我们常用的：

```rust
println!("array[2] = {}", array2[2]);
```

这当然会打印“hello jack”，但如果访问一个不存在的索引呢？

```rust
println!("array[6] = {}", array2[6]);
```

此时会panic：

```rust
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 6', src/main.rs:14:31
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

还需要非常注意的是，之前也提到过，Vec不允许自己的元素被转移走，即以下代码将编译不通过：

```rust
let take = array2[1];
```

因为首先Rust不允许悬空指针的存在，其次如果Vec去维护中间哪一个元素处于悬空指针又实在不合常理，毕竟Vec就是Vec，即向量，向量不应该感知这些东西。因此如果有人打算通过索引访问去取走Vec的资源，那么就会被编译器报错。



### 删除操作

如果想删去Vec的元素，推荐用的是pop()方法，但注意pop() 方法只能从Vec的最后一个元素开始删除，毕竟前面也说了，试图从中间开始插入或删除一个Vec会引起数据移动操作，这是非常低效的。

例如我们pop出最后一个元素：

```rust
assert_eq!(array2.pop(), Some("hello jack".to_string()));
assert_eq!(array2.len(), 2);
```

可以看到，pop()参数返回的是一个Option\<T\>，也就是如果array2是空的，那么就会返回None。这里返回的是Some(String)类型。pop之后，array2就只有两个元素在里面了。

但要注意，如果用for in的方式访问array2且没有使用引用的话，还是会把里面的资源转移走的：

```rust
    for s in array2 {
        println!("{}", s);
    }
    assert_eq!(array2.pop(), Some("hello jack".to_string()));
```

以上代码编译不通过，因为第1行for in语句中，s会逐次获得array2的资源，第4行再去访问array2的资源会编译失败。



### Vec内部结构探究

Vec实际上内部是封装了一个tuple结构，tuple中第一个是指针，指向T资源，第二个是长度len，第三个是容量capacity。Vec保证当容量不足时就会自动懂扩容，因此自动扩容的出现点出现在所需内存大于capacity时。把上面的array2结构画出来，会是这样（String结构这里简化了，关于String结构后一节会讲，这里只关注Vec）：
