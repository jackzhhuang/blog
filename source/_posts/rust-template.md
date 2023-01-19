---
title: Rust的模版和trait编程
date: 2023-01-17 13:13:33
toc: true
tags:
- Rust
categories:
- Rust
---

要开始学习Rust比较高级的内容了。

今天要讲的是Rust的模板编程。模板编程实际上并不少见，不论C++还是Java，只要使用容器，基本都离不开模板。Rust的模板和C++很相似，都是编译时根据模板代码生成实际的（concrete）代码然后再编译成二进制。

那么，今天就好好看看Rust的模板编程吧。

<!--more-->

## 模板

### 函数模板

假设我们需要做查处整型和字符型数组中，最大的那个元素，那么需要两个函数分别处理：

```rust
fn find_largest_int(array: &[i32]) -> &i32 {
    let mut largest = &array[0];
    for i in array {
        if largest < i {
            largest = i;
        }
    }
    largest
}

fn find_largest_char(array: &[char]) -> &char {
    let mut largest = &array[0];
    for i in array {
        if largest < i {
            largest = i;
        }
    }
    largest
}
```

很明显，两个函数都差不多，更抽象的写法是使用模板，即不管什么类型，都执行加法运算：

```rust
fn find_largest_template<T>(array: &[T]) -> &T {
    let mut largest = &array[0];
    for i in array {
        if largest < i {
            largest = i;
        }
    }
    largest
}
```

find_largest_template\<T\>的T表示T是一个种类型的模板，即模板声明， array: &[T]则表示array是一种类型的slice引用，返回值这是一种类型的引用。具体类型在编译确定的代码时会确定。目前为止没错。

但在第4行，会报一个错：

```rust
   |
26 |         if largest < i {
   |            ------- ^ - &T
   |            |
   |            &T
   |
help: consider restricting type parameter `T`
   |
23 | fn find_largest_template<T: std::cmp::PartialOrd>(array: &[T]) -> &T {
   |                           ++++++++++++++++++++++

For more information about this error, try `rustc --explain E0369`.
error: could not compile `greeting` due to previous error
```

意思是，需要确保调用者知道，这个模板T类型需要实现std::cmp::PartialOrd这个trait（partial order），即这个模板类型需要能做<，>，<=和>=的运算。这个用法相当于函数对调用方的要求，限制。也就是trait bond，即想调用这个函数，调用方必须保证模板类型能满足指定的trait。我们按照编译器的指示，加上对应的trait bond后，编译器检查无误，可以通过编译并正常运行了：

这个例子我们可以看出三点：

1、通过函数入参来推断模板类型；

2、返回值也可以用于模版；

3、函数设计方可以对模板进行trait bond设计，保证使用方知道这个trait需要满足什么特性（trait）。



### struct模板

和函数模板差不多，我们在struct名称前加上模板声名就可以声明一个模板struct了，后面的字段可以定义对应的类型字段：

```rust
#[derive(Debug)]
struct Coord<T> {
    x: T,
    y: T,
}

#[derive(Debug)]
struct Point<T, U> {
    number: T,
    coord: U,
}
```

在使用时，我们也和函数模板一样（函数模板通过入参来推断T），在初始化struct的字段时，推断出各个字段的类型：

```rust
    let postion = Point {
        number: 100,   // 表明Point的T是i32
        coord: Coord { // 表明Point的U是Coord
            x: 23,     // 表明Coord的T是i32
            y: 84,     // 表明Coord的T是i32
        },
    };
```

类似的还有enum的模板，这在之前学习Result和Optional的时候遇到过，不多说了。



### 方法的模板

方法的模板稍显复杂，先看一个简单的例子，例如，我们给前面的例子中Coord\<T\>添加一个方法用以计算坐标到o点的距离，这个方法是针对T为f64的特化版本：

```rust
impl Coord<f64> {
    pub fn distance_to_o(&self) -> f64 {
        (self.x.powf(2.0) + self.y.powf(2.0)).sqrt()
    }
}
```

可以看到如果要写特化，只需要在impl Coord后面实例化T即可。

当然也可以不特化，但如果需要支持运算就要加上trait bond，这里用打印x和y来举例：

```rust
impl<T: std::fmt::Display> Coord<T> {
   pub fn show(&self) {
        println!("x = {}, y = {}", self.x, self.y);
   } 
}
```

可以看到相对特化版本，泛型版本需要在impl\<T>中对T进行trait bond指定，让调用方知道，想使用这个方法，T必须实现了std::fmt::Display，因为方法实现中调用了println!宏。

总结来说，特化需要放在struct名中指定具体类型，而泛型需要放在impl中指定trait bond。

我们还可以对泛型为T的struct加上泛型为U的方法：

```rust
#[derive(Debug, Clone, Copy)]
struct Coord<T, U> {
    x: T,
    y: U,
}

impl<T: Copy, U> Coord<T, U> {
   pub fn mix<P, Q: Copy>(&self, other: &Coord<P, Q>) -> Coord<T, Q> {
        Coord { x: self.x, y: other.y }
   } 
}


fn main() {
    let a = Coord {
        x: 30,
        y: 50,
    };
    let b = Coord {
        x: 3.14,
        y: 2.718,
    };

    let c = a.mix(&b);

    println!("c = {:#?}", c);
}
```

上面就是把两个泛型类型不一样的Coord合成第三种泛型类型不一样的过程，可以看到，impl\<T: Copy, U\>可以指定对某一类struct进行添加方法，即对struct这一层的模板声明（类似C++的tempate\<class T\>），而 mix<P, Q: Copy>则是对方法的参数进行某一类的指定，即对方法这一层的声明，最后在返回值的地方返回混合类型的Coord\<T, Q\>。

这里还需要注意的时候我们在一些类型后面加上了Copy trait，因为mix的入参都是引用，但返回的Coord\<T, Q\>不是，所以值必须通过拷贝来赋值而不是直接赋值。否则，如果是直接赋值，也就是返回值的T和Q类型分别是引用的话，那么引用入参可能生命周期不如返回参数长，返回值中的字段引用的数据资源可能已经被销毁了，造成悬空指针。



## trait编程

### 什么是trait

trait英文原意是特性的意思，这里有一个很好的比喻，可以快速理解计算机编程语言的trait的特性和用法。我们怎么知道某个东西是一只兔子？兔子的trait是：

1、耳朵长；

2、眼睛红；

3、跑起来像跳；

4、吃胡萝卜。

假设我们总结出来兔子的trait如上四条，那么，如果遇到满足以上四条trait的东西，我们就认为它是兔子，哪怕我把我的MacBook笔记本按照这四个trait伪装起来，那我的MacBook笔记本就是一只兔子。因为实际上，我们关心并不是到底是兔子还是笔记本，而是它是否有这些trait。

也就是说，其实我们并不关心某个东西是什么？我们只关心的是某个东西是否满足某个或者某些trait，满足了，那就行了，具体是什么，其实关系不大。

这就是我们在面向对象编程里面说的，我们要面向接口编程。这里，就是说成：我们要面向trait编程。



### 实现trait

了解了什么是trait，那么就知道trait是做什么的了，其实就是给各种类型定义行为。

```rust
struct Article {
    headline: String,
    location: String,
    author: String,
    content: String,
}

struct Tweet {
    username: String,
    content: String,
    reply: bool,
    retweet: bool,
}

trait Summary {
    fn summarize(&self) -> String;
} 

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("writen by {}, summary: {}", self.author, self.headline)
    }
}

impl Summary for Tweet {
     fn summarize(&self) -> String {
        format!("writen by {}, summary: {}", self.username, self.content)
    }
   
}

fn main() {
    let a = Article {
        headline: String::from("a story about rust"),
        content: String::from("this is will be a long long story you've never head before"),
        author: String::from("jack"),
        location: String::from("cn"),
    };

    let t = Tweet {
        username: "rose".to_string(),
        content: "coding makes my day".to_string(),
        reply: true,
        retweet: false,
    };

    println!("{}", a.summarize());
    println!("{}", t.summarize());
}
```

上面的代码用了一个简单的例子，Article和Tweet有不同的字段，同时定义了一个trait Summary，然后针对Article和Tweet分别写出了不同的summarize方法实现。这样，不管Article和Tweet字段怎么样，他们都有Summary这个trait，都可以调用这个trait下的方法。自然有人问，各自定义自己的summarize方法不就行了吗？这个例子的确如此，但假设我们有一个函数去专门处理Summary，做这个事情的人不想关心具体是什么东西，也许是Article，或者是Tweet，甚至可能是一只兔子？总之，对他来说，只要是Summary就行，想使用他的服务的人去想办法把自己变成Summary吧，也就是，只要有Summary这个trait，他的函数就能处理：

```rust
fn collect_summary(s: &dyn Summary) {
    println!("{}", s.summarize());
}
```

 collect_summary这个函数入参是&dyn Summary，是一个引用，其次dyn是表示运行时动态（dynamic）类型，即编译时不知道Summary背后的东西是什么，需要运行时才能确定，Summary就是我们的trait。加起来表示：collect_summary函数接受一个Summary trait动态引用。

这样，编写trait和对trait进行再封装再处理的两个人可以解藕编程了，他们之间只需要约定好trait即可。



### dispatch机制

顺带一提，这里把dyn改成impl也可以正常运行，那么dyn和impl有什么区别呢？还有，为什么dyn前面那里需要用引用 & 呢？本小节讲明白。

其实这里是Rust的动态运行机制，当涉及到多态编程时，Rust有一种机制来决定到底应该用哪个具体类型来执行，这个机制叫“dispatch”。而dispatch机制分为静态和动态两种，静态dispatch，即编译时期就知道具体是哪个类型了，动态dispatch，则需要在运行时期才能知道是什么类型。

impl trait则是静态dispatch，也即，Rust编译器在编译的时候就会知道都有哪些类型会使用impl trait，并把这些impl trait代码都生成好，运行时直接调过去就行，运行时不需要判断。例如上面的例子中，我们修改collect_summary函数为：

```rust
fn collect_summary(s: impl Summary) {
    println!("{}", s.summarize());
}
```

 在调用的时候，直接传实现了Summary trait的对象指针进去即可：

```rust
    let a = Article {
        headline: String::from("a story about rust"),
        content: String::from("this is will be a long long story you've never head before"),
        author: String::from("jack"),
        location: String::from("cn"),
    };

    let t = Tweet {
        username: "rose".to_string(),
        content: "coding makes my day".to_string(),
        reply: true,
        retweet: false,
    };

    collect_summary(a);
    collect_summary(t);
```

因为是静态dispatch，Rust编译器需要在编译的时候就确定到底谁调用了collect_summary，当看到collect_summary(a)和collect_summary(t)会分别生成对应的函数：

```rust
fn collect_summary(s: Article) {
    println!("{}", s.summarize());
}

fn collect_summary(s: Tweet) {
    println!("{}", s.summarize());
}
```

于是这样就实现了编译器的多态，即静态dispatch。（额外提示，此处a和t会失去所有权）

而动态dispatch，即dyn trait，编译时期编译器无需关心具体是谁调用了collect_summary，它只需要记得，collect_summary接受一个实现了Summary trait的类型做为参数即可，由于编译时期无需关心collect_summary的入参具体类型，也就不知道这个入参的大小，即sizeof大小，此时若不是用引用，那么在不知道sizeof大小的情况下，是无法生成collect_summary函数的代码的（因为要确定内存栈大小）。因此，此时必须使用&dyn。编译器也不会生成各种类型入参的collect_summary版本，这些事情都要放到运行时期才确定。



### trait bond

trait bond前面已经用到，就是对模板 T 进行有所限制，只有满足特定trait的类型才能实例化。如果需要多个trait限定，就用 + 号来增加trait限定。比如，给Coord增加一个方法，只有实现了对比trait和现实trait的类型才能实例化：

```rust
use std::fmt::Display;

struct Coord<T> {
    x: T,
    y: T,
}

trait ShowMax {
   fn show_max(&self) -> String; 
}

impl<T: Display + PartialOrd> ShowMax for Coord<T> {
    fn show_max(&self) -> String {
       if self.x >= self.y {
            return format!("{}", self.x);
       }

       format!("{}", self.y)
    }
}

fn main() {
    let c = Coord {
        x: 30,
        y: 18,
    };
    println!("max = {}", c.show_max());
}
```

上面的代码中可以看到，我们对实现ShowMax这个trait进行了trait bond限定，只有能做到Display（进行格式化显示）和PartialOrd（进行大小判断）的类型才能调用。trait bond有助于编译器检查类型是否满足条件。



### where关键字

trait bond除了可以写在简括号里面，还可以放在函数或者struct名字后面，用where引导出trait bond，比如上面的例子中，可以用where改写成这样：

```rust
impl<T> ShowMax for Coord<T> 
    where T: Display + PartialOrd {
    fn show_max(&self) -> String {
       if self.x >= self.y {
            return format!("{}", self.x);
       }

       format!("{}", self.y)
    }
}
```





## 总结

本章内容比较抽象，有一定的编程经验的人才能看懂，总结几个细节小点：

1、注意模板声明的位置。想使用模板必须先声明，函数模板在函数名称后面声明，struct在struct类型名称后面声明，方法模板则在impl后面声明；

2、struct模板和方法模板可以是两个不同的维度各自声明自己的模板；

3、对trait编程，实现模块间解藕；

4、区分impl trait（静态dispatch，编译时确定）和dyn trait（动态dispatch，运行时确定）两种dispatch机制，已经此时引用的含义；

5、使用trait bond保证模板能有某种trait，帮助编译器检查前置条件。

