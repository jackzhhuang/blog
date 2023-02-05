---
title: Rust的容器
date: 2023-01-12 22:42:05
toc: true
tags:
categories:
- Rust
---

今天熟悉一下Rust的常见容器和其用法。当然本节介绍的容器只是Rust众多数据结构中的一角，只能算是起个头，未来使用容器时，都应该多翻翻文档，并且每次都应该想想：

1、用来做什么？（是数组还是字符串，是需要哈希还是需要有序）

2、对数据访问有什么特别的要求？（顺序访问还是随机访问，要经常插入删除还是多是遍历）

3、其它各种先决条件。

总之，选择容器一定要看上下文的需求，选择最合适的容器去解决问题。同时，多看文档，多翻文档。

现在我们就从最常用的容器开始吧。

<!--more-->

## 1. 数组Vec

数组Vec是最常用的容器，它非常类似C++的std::vector，底层数据结构也是一块连续的内存，内存不足时，会扩大，并把老数据拷贝到新内存上，因此，非常适合顺序访问和索引访问，非常不适合有删除和插入的场景，因为此时暗含着内存移动的低效操作。

### 1.1 高效初始化

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



### 1.2 顺序访问

Vec是最适合顺序访问的了，我们可以一个个枚举Vec里面的值：

```rust
    for s in &array2 {
        println!("{}", s);
    }
```

注意，访问array2时我们用了引用，这样访问里面的String对象时，也都变成了引用，这个规则struct也是这样。因为用了引用，后续array2还是有效的指针。



### 1.3 索引访问

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

### 1.4 插入操作

#### 1.4.1 insert()方法

相对于push操作，每一次的insert() 的操作就需要大量移动移动数据了，push只有超过capacity的时候才会产生数据转移，而insert每一次都要把需要插入的位置的后面元素往后挪动一个位置，空出来后给新元素放入：

```rust
    let mut array1 = vec![192, 31, 231, -12, 3];

    println!("before insert, array1 = {:?}", array1);

    array1.insert(1, 88);


    println!("after insert, array1 = {:?}", array1);
```

如果插入操作真的需要很频繁，那么就应该用 LinkedList\<T\>。



### 1.5 删除操作

#### 1.5.1 pop()方法

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

#### 1.5.2 swap_remove () 方法

如果真的需要删掉其中某一个元素，那么可以使用swap_remove()。它的入参是一个index，即需要删掉的元素的索引。为了避免数据移动，Vec的删除策略是这样的：把最后一个元素的值和需要删掉的index交换，并把Vec的长度减1:

```rust
    let mut array1 = vec![192, 31, 231, -12, 3];

    println!("before swap_remove, array1 = {:?}", array1);

    array1.swap_remove(1);

    println!("after swap_remove, array1 = {:?}", array1);
```

输出如下：

```rust
before swap_remove, array1 = [192, 31, 231, -12, 3]
after swap_remove, array1 = [192, 3, 231, -12]
```

可以看到，我们打算删掉31，于是传入它的index=1，Vec把末尾的3交换到31上，然后长度减1。同时，swap_remove还会把31返回出来，虽然这里程序没有提现。



### 1.6 Vec内部结构探究

Vec实际上内部是封装了一个tuple结构，tuple中第一个是指针，指向T资源，第二个是长度len，第三个是容量capacity。Vec保证当容量不足时就会自动懂扩容，因此自动扩容的出现点出现在所需内存大于capacity时。把上面的array2结构画出来，会是这样：

![Vec内部](https://www.jackhuang.cc/svg/Vec.svg)

可以看到，上图蓝色的三个元素就是Vec内部主要封装的三个熟悉，用tuple表示为 (p, 3, 6) ，即p表示指向一组连续的内存，放String数组，3则是数组的元素个数，6则表示array2的容量，表示还可以再存放6 - 3 = 3个String对象，且保证在此之前不会触发数据移动操作。

String数组后面的灰色部分是未初始化区域，也就是6 -3 = 3这个部分。关于未初始化区域，可以详细看这里：https://doc.rust-lang.org/std/mem/union.MaybeUninit.html 。



## 2. 字符串String

### 2.1 字符串的连接

两个字符串连接是最常见的编程场景，最显而易见的就是使用String的push_str和push两个方法，前者入参是一个字符串，后者入参是一个字符：

```rust
    let mut s1 = String::from("你好");
    s1.push('，');
    s1.push_str("世界");

    println!("s1 = {}", s1);
```



另外一个方法，就是直接用 + 号来实现字符串的拼接，需要注意的是，+ 号方法，使用的是self而不是&self，也就是说，会有所有权转移：

```rust
fn add(self, s: &str) -> String {
```

还有需要注意的是，s这个参数是&str，也就是它接受一个引用，也就是被加的字符串不会有所有权转移，例如：

```rust
    let mut s1 = String::from("你好");
    let s2 = String::from("，");
    let s3 = String::from("世界！");

    s1 = s1 + &s2 + &s3;

    println!("s1 = {}", s1);
    println!("s2 = {}", s2);
    println!("s3 = {}", s3);
```

上面的代码中，s1 + &s2 + &s3会使得s1丢失所有权，所幸又还给了s1。这里Rust之所以这么干，其实是避免了拷贝，其内部只是通过引用拿到s2和s3的内容，拷贝到s1的后面，避免了拷贝一块新的区域来存放s1。



### 2.2 字符串和字节

String是和Vec类似的，内部结构也是一个tuple，同样放着p指向字符串，另外两个是字符串的长度和容量。这里不展开讲太多String的功能方法，稍有编程经验的人都应该知道String会提供哪些典型的方法，具体可以参考官方文档。

和其它语言不同，需要特别注意的是，Rust的String对象是一个UTF8格式的，这就意味着一个问题，就是如果我们索引访问一个String对象，会是返回什么？

如果String内容是“hello world!”，那么[0]是否应该是h，如果是h，那么它的大小就是一个字节。

如果String内容是“你好，世界！”，那么[0]是否应该是“你”，如果是“你”，那么它的大小就是三个字节。

那么到底索引访问一个String要返回多少个字节呢？Rust觉得这是一个模糊的问题，于是拒绝编译这种代码：

```rust
    let s = String::from("你好，世界！");

    let c = s[1];
```

编译器报错如下：

```rust
 --> src/main.rs:4:13
  |
4 |     let c = s[1];
  |             ^^^^ `String` cannot be indexed by `{integer}`
```

Rust 需要使用方明确知道自己在干什么，是要返回一个字符，还是要返回某个字节，所以，如果想拿到某个字符，就明确的告诉String，我要以字符方式访问你：

```rust
    let s = String::from("你好，世界！");

    let mut c = s.chars();

    println!("c = {:?}", c.nth(1).unwrap());
    println!("s = {:?}", s);
```

chars() 方法返回一个迭代器，即我们熟悉的iterator，可以看到做为迭代器，它是有状态的，即封装了一个当前所处位置的指针，因此需要加上mut，这样后面的nth() 方法才可以得以调用，因为nth() 方法会把当前位置往前移动，这相当于修改了状态。

nth返回Option\<Char\>，调用unrawp() 方法可以拿到里面的字符。

我们当然也可以用字节访问String对象，但这样，打印出来的就不是字符形式了，而是字节数字：

```rust
    let s = String::from("你好，世界！");

    let mut b = s.bytes();

    println!("c = {:?}", b.nth(1).unwrap());
    println!("s = {:?}", s);
```

输入出如下：

```rust
c = 189
s = "你好，世界！"
```

189其实是“你”这个字符的第二个字节数。



### 2.3 比较两个String

Rust在这方面要求就比较简单，只需要 == 去比较就好了：

```rust
    let s1 = String::from("你好世界!");
    let s2 = "你好世界!";

    if s1 == s2 {
        println!("they are same!");
    } else {
        println!("they are different!");
    }
```

输出为：

```rust
they are same!
```

虽然s1是String对象，s2是原生字符串，s1是对象的指针，s2是一个引用，但Rust还是会去判断这两个字符串是否内容相等。



### 2.4 Rust的String不简单

Rust的String并不简单，这里只是主要提了最重要的一点，即String需要我们时刻知道我们到底是造操作字符还是字节，未来遇到String的使用，还是需要多翻阅文档，这里就不去一句句翻译文档内容了。



## 3. 哈希HashMap

hash也是日常中最常用的数据结构，Rust的HashMap使用了SpiHash算法，避免了哈希碰撞的攻击，详见：https://en.wikipedia.org/wiki/SipHash 。当然，其效率就有所下降。

这里主要讲的是我们常使用的方法，第一个当然是插入了。

### 3.1 插入

```rust
    use std::collections::HashMap;
    let mut h1 = HashMap::new();
    h1.insert(String::from("hello world!"), 10);
    h1.insert(String::from("hello rust!"), 20);
    h1.insert(String::from("hello jack!"), 30);

    println!("h1 = {:#?}", h1);
```

可以看到，首先HashMap和Vec不一样，需要我们显式引入，其次，它的key和value类型可以在insert的时候指定，当然一旦指定好类型，之后都不允许更改了。

注意，由于我们没有使用引用，因此，一旦对象呗插入到HaspMap中，HashMap就有了对象的所有权，如果使用引用，那么我们就必须保证所插入的数据的生命周期大于HashMap对象的生命周期，这在后一节讲到。



### 3.2 查找

使用get方法可以获得一个Optional\<T\>，如果不存在，返回None，否则返回Somel\<T\>：

```rust
println!("h1[\"hello rust!\"] = {}", h1.get("hello rust!").copied().unwrap_or(0));
```

这里调用了copied方法，因为返回的是Optional\<&i32\>，最后调用unwrap_or() 方法，并且提供了若是None，则返回0。

这里的错误处理下一节会专门讲到。

当然除了用ge() t方法，还是使用for语句去遍历所有的HashMap数据：

```rust
    for (key, value) in h1 {
        println!("{} = {}", key, value);
    }
```



### 3.3 更新

如果我们在插入同样的key会发生什么呢？比如再插入一条key为"hello rust!"，value为80的数据：

```rust
    use std::collections::HashMap;
    let mut h1 = HashMap::new();
    h1.insert(String::from("hello world!"), 10);
    h1.insert(String::from("hello rust!"), 20);
    h1.insert(String::from("hello jack!"), 30);
    h1.insert(String::from("hello rust!"), 80);

    println!("h1[\"hello rust!\"] = {}", h1.get("hello rust!").copied().unwrap_or(0));
```

 输出为：

```rust
h1["hello rust!"] = 80
```

可以看到，原来的20被80给覆盖了，因此，插入同样的key会产生覆盖结果。

但如果我们并不想去覆盖，如果已经存在就不要去动它了，怎么办呢？此时应该调用Entry() 方法，Entry() 方法返回一个枚举，如果有值，则返回那条记录的引用，否则返回新的记录的引用，新记录的枚举中，调用它的or_insert() 方法就可以插入新值：

```rust
    use std::collections::HashMap;
    let mut h1 = HashMap::new();
    h1.insert(String::from("hello world!"), 10);
    h1.insert(String::from("hello rust!"), 20);
    h1.insert(String::from("hello jack!"), 30);
    h1.entry(String::from("hello rust!")).or_insert(80);

    println!("h1[\"hello rust!\"] = {}", h1.get("hello rust!").copied().unwrap_or(0));
```

此时输出为：

```rust
h1["hello rust!"] = 20
```

20没有再被80覆盖。

如果我们要针对老值来更新新值呢？比如统计“how many word in this word ?” ：

```rust
   use std::collections::HashMap;

    let s = "how many word in this word ?";

    let mut h = HashMap::new();

    for key in s.split_whitespace() {
        let count = h.entry(key).or_insert(0);
        *count += 1;
    } 

    println!("h = {:#?}", h);
```

输出为：

```rust
h = {
    "in": 1,
    "this": 1,
    "how": 1,
    "?": 1,
    "many": 1,
    "word": 2,
}
```

可以看到由于word这个词已经存在，所以，第二次调用的时候，entry()  方法返回了老的记录的音乐，此时若调用or_insert()  方法并不会更新，并且返回value的引用，后面的*count += 1对word这个词进行了 + 1 统计。

以上就是我们常用的HashMap方法，当然，还是那句话，我们使用容器的时候应该多考虑开篇说的两个问题，即什么场景使用，会怎么使用，并且应该常翻阅文档，在里面看看有没有合适的方法供我们选择。

