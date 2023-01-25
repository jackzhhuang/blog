---
title: 闭包和迭代器
date: 2023-01-25 14:44:32
tags:
- Rust
- Closure
- Iterator
categories:
- Rust
---

闭包和迭代器很多语言都有，Rust也不例外，并且，数量掌握闭包和迭代器是写出一手好代码的必要因素。今天就把这两个概念拿出来说说。

<!--more-->

## 闭包（closure）

闭包，即closure，简单说即：

1、匿名函数；

2、可以当作value赋值，灵活调用；

3、闭包的入参和出参除了定义闭包的时候可以指定，还可以由编译器推断（这样更简洁）。

以下是最简单的例子：

```rust
fn main() {
    let v: Vec<String> = vec!["hello world!".to_string(), 
                              "hello rust!".to_string(),
                              "hello jack!".to_string(),];

    v.iter().for_each(|x| println!("print in closure: {}", x));
}
```

 这里的闭包即“|x| println!("print in closure: {}", x)”，这是一个匿名函数，入参为x，编译器会推理为String，根据代码的实现，没有返回参数，只是简单的打印了String的值。

关于闭包这里不在对其定义阐述更多基本的内容，只需要知道，Rust的闭包入参和出参都是可以依靠编译器推断的就行，关键是Rust的闭包相对于其它语言需要留意的地方。



### 是move还是引用？

闭包最大的特点就是可以capture上下文变量，那么，这个capture在Rust中是move还是引用呢？答案是：引用。例如：

```rust
    let list = vec![1, 2, 3];
    let show = || {
        println!("list in side the colosure: {:?}", list);
    };

    show();
    println!("list after calling the show: {:?}", list);
```

上面的例子中，show这个闭包不接受任何参数，但capture了上面的list，show被调用后，list所有权没有被show拿走，依然还在，第7行正常打印list。

那么，如果闭包就是像move走所有权呢？需要在闭包的前面加上move：

```rust
    let list = vec![1, 2, 3];
    let show = move || {
        println!("list in side the colosure: {:?}", list);
    };

    show();
    println!("list after calling the show: {:?}", list);
```

此时编译出错：

```rust
error[E0382]: borrow of moved value: `list`
  --> src/main.rs:10:51
   |
4  |     let list = vec![1, 2, 3];
   |         ---- move occurs because `list` has type `Vec<i32>`, which does not implement the `Copy` trait
5  |     let show = move || {
   |                ------- value moved into closure here
6  |         println!("list in side the colosure: {:?}", list);
   |                                                     ---- variable moved due to use in closure
...
10 |     println!("list after calling the show: {:?}", list);
   |                                                   ^^^^ value borrowed here after move
```

因为list的所有权被show拿走了，最后的print对list的访问将会报错。



### mut闭包

默认情况下，和Rust基本特性一样，闭包对上下文的capture都是immutable的，如果需要在闭包中更改变量，除了被capture的上下文需要是mut的以外，闭包也需要声明为mut：

```rust
    let mut list = vec![1, 2, 3];
    let mut more = || {
        list.push(4);
    };

    more();
    println!("{:?}", list);
```

上面这个例子中，list和more都需要mut才能编译通过。



### FnOnce， FnMut和Fn

闭包结合Rust的所有权特性，会对闭包的调用有所限制。闭包的trait有三个：FnOnce，FnMut和Fn。

FnOnce，顾名思义，表示因为有所有权转移，该闭包只能被调用一次。

FnMut，Mut即表示改变的意思，意思是该闭包会修改上下文变量，但不会产生所有权转移。

Fn则比较佛性，即不修改上下文变量名，也不产生所有权转移。

看下面这个例子：

```rust
#[derive(Debug)]
struct Rectangle {
    height: u32,
    width: u32,
}

fn main() {
    let r1 = Rectangle {
        height:40,
        width: 50,
    };
    let r2 = Rectangle {
        height:70,
        width: 20,
    };
    let r3 = Rectangle {
        height:100,
        width: 60,
    };

    let mut list = vec![r1, r2, r3];
    let s = String::from("return once");
    let mut operation = Vec::new();

    list.sort_by_key(|r| {
        operation.push(s);
        r.width
    });

    println!("the list is {:?}", list);
}
```

本意是调用一次sort_by_key方法就会push一次字符串，但编译不通过的是，sort_by_key的入参中，闭包需要是FnMut，可上面的代码中，

```rust
|r| {
        operation.push(s);
        r.width
    }
```

只能被调用一次，因为第二次s已经没所有权了，即此时闭包的trait是FnOnce属性的，打开sort_by_key方法的代码可以发现的确是需要FnMut的：

```rust
  #[cfg(not(no_global_oom_handling))]
    #[rustc_allow_incoherent_impl]
    #[stable(feature = "slice_sort_by_key", since = "1.7.0")]
    #[inline]
    pub fn sort_by_key<K, F>(&mut self, mut f: F)
    where
        F: FnMut(&T) -> K,
        K: Ord,
    {
        merge_sort(self, |a, b| f(a).lt(&f(b)));
    }
```

闭包是迭代器最常用的工具，而迭代器又是批处理集合最常用的方法，下面我们就看看Rust的迭代器是怎么用的吧。



## 迭代器（iterator）

迭代器是Rust处理集合最最最常用的方法，可以说没有之一。如果依然使用for语句去处理集合，一来不够Rust，二来效率不如迭代器快，因为迭代器是零成本抽象实现。

关于零成本抽象有很多介绍，但很多人不太理解，这里尝试用最简单的人话说：即只仅仅实现需要的代码，无需额外的代码（成本）。

再简单一步说，即人类目前设计出最抽象的代码，以至于你不可能再写出更抽象的了（如果可以，请提交PR），因为已经是最抽象的代码，已经没有简化优化的空间，因此效率是最高的。

迭代器就是高度零成本抽象的代码，我们应该尽可能使用迭代器去处理集合，而不是写for语句。

Rust中迭代器是一个trait，基本定义如下：

```rust
pub trait Iterator {
    /// The type of the elements being iterated over.
    #[stable(feature = "rust1", since = "1.0.0")]
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
  ...
```

其主要的方法是next，每次调用next迭代器都会返回当前元素Some(T)，并且跳到下一个元素，直到所有元素被完全枚举，此时返回None。如果自己定义一个集合也想有迭代器trait，那么也需要去实现这个next方法。

从上面看出，迭代器是有状态的，即它会记录目前在集合中的位置：

```rust
    let v = vec![
        String::from("hello world"),
        String::from("hello rust"),
        String::from("hello jack"),
    ];

    let mut iter = v.iter();

    assert_eq!(iter.next(), Some(&String::from("hello world")));
    assert_eq!(iter.next(), Some(&String::from("hello rust")));
    assert_eq!(iter.next(), Some(&String::from("hello jack")));
    assert_eq!(iter.next(), None);
```

那么，当手里拿着一个集合时，如果需要逐个处理，我们应该首先把它变成迭代器，Rust的集合有三个不同的迭代器变化方法，分别对应三个用途，即：iter，into_iter和iter_mut。



### iter，into_iter和iter_mut

iter是把集合转成引用迭代器，即访问集合元素时以引用的方式访问，不会有所有权转移。例如：

```rust
   let v = vec![
        String::from("hello world"),
        String::from("hello rust"),
        String::from("hello jack"),
    ];

    v.iter().for_each(|x| println!("{}", x));

    println!("all v = {:?}", v);
```

上面第7行 x 其实是&String，因此第9行访问 v 时，v 依然时有效的集合。顺便所一下，for_each会拉起对集合的处理流程，即对每一个元素都以参数的方式传入 for_each 中的闭包，即上面的 x。

如果改成into_iter方法，那么 x 就会变成 String，导致 v 中元素对 String 所有权丢失（都跑 x 去了）：

```rust
17 |     v.into_iter().for_each(|x| println!("{}", x));
   |       ----------- `v` moved due to this method call
...
26 |     println!("all v = {:?}", v);
   |                              ^ value borrowed here after move
```

而 iter_mut 即上面的 x 为可变引用（&mut String），注意此时集合本身也必须是可变的才行。这里不多举例了。

稍微总结一下：

iter方法：转为引用访问的迭代器，因此没有所有权转移；

into_iter方法：转为所有权转移的迭代器，因此集合会丧失对所有元素的所有权；

iter_mut：转为可变引用访问的迭代器，因此也没有所有权转移，且还可以修改元素。



### map和collect

前面演示了迭代器的 for_each 方法，它只是遍历一遍元素，实际上我们不仅可以遍历一遍元素，还可以把这些被处理的元素以新的集合返回出来，例如map和collect组合，我们先用map：

```rust
    let mut v = vec![
        String::from("hello world"),
        String::from("hello rust"),
        String::from("hello jack"),
    ];

    v.iter_mut().map(|x| x.push_str("!"));

    println!("all v = {:?}", v);
```

上面的例子中 map 的 x 就是&mut String，我们对 x 进行追加一个 “!”  操作，它作为返回值返回给新的迭代器，但这里并没有把新的迭代器写出来，编译执行以上代码，发现 v 并没有变化：

```rust
all v = ["hello world", "hello rust", "hello jack"]
```

 这是怎么回事呢？因为和 for_each 不同，map 是个偷懒的函数，如果没有新的迭代器去装返回值，那么它就不会被触发，另外，新的迭代器类型是靠返回值类型来推断出来的，我们可以用其它迭代器来装：

```rust
    let mut v = vec![
        String::from("hello world"),
        String::from("hello rust"),
        String::from("hello jack"),
    ];

    let list: LinkedList<_> = v.iter_mut().map(|x| {
        x.push_str("!"); 
        x.clone()
    }).collect();

    println!("all v = {:?}", v);
    println!("all list = {:?}", list);
```

这里用了 LinkedList\<\_\> 来装。调用迭代器的collect方法会迫使迭代器去产生新的迭代器进而产生新的集合。

为什么这里 LinkedList\<\_\> 省略了模板类型呢？用了下划线 _ 来代替，因为编译器可以推理出下划线 _ 即 String，我们可以省略不写太多字母。

此外，我们的map中的闭包返回了 x.clone()，即&mut String的 clone，这是因为如果返回 x，即 LinkedList\<\_\>为LinkedList<&mut String>，最后两句print宏会产生歧义，即 v 是immutable引用，而v是mutable引用，Rust不允许对同一个对象即有mut又有immut引用。

总之，我们可以用迭代器产生新的迭代器，其产生映射可以用 map 函数来生成新的集合元素，map返回新的迭代器。

而用迭代器的 collect方法把迭代器重新变回为集合，集合类型可以用返回值推导出来。这是最常用的方法了。

除了map，还有不少其它引射方法，比如filter，其输入的闭包若返回true则进入到新的迭代器中：

```rust
    let v = vec![1, 2, 3, 4, 5, 6, 7];
    let evens: Vec<_> = v.into_iter().filter(|x| x % 2 == 0).collect();

    println!("evens = {:?}", evens);
```

输出为偶数集合：

```rust
evens = [2, 4, 6]
```

