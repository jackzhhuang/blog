---
title: Rust中的enum和match
date: 2023-01-08 19:25:24
toc: true
tags:
- Rust
categories:
- Rust
---

今天就来熟悉一下Rust的enum和match语法吧。

<!--more-->

## 1. 定义一个enum

枚举类型从来不缺席任何一个稍微高级的语言。Rust也不例外，定义一个枚举只需要enum关键字即可，比如我们定义IP地址类型，其有两种格式，V4和V6版本：

```rust
#[derive(Debug)]
enum IpAddressType {
    V4,
    V6
}
```

那么，如果有一个地质结构体，就可以用它来标识一个地址是哪一个类型的，使用方法也和其他语言差不多：

```rust
#[derive(Debug)]
enum IpAddressType {
    V4,
    V6
}

#[derive(Debug)]
struct IpAddress {
    address_type: IpAddressType,
    address: String
}

fn main() {
    let ip = IpAddress {
        address_type: IpAddressType::V4,
        address: "127.0.0.1".to_string()
    };

    println!("ip = {:#?}", ip);

}
```

可以看到，ennum此时的用法和C++差不多一样，几乎没有特殊之处，下面，就说一下Rust的enum类型带来的更多特性。



### 1.1 给enum绑定值

上面的例子，IpAddressType定义了V4和V6两个类型，要使用它们去标识一个IP需要和String类型一起使用，即IpAddressType标识String的内容是V4还是V6，enum实际上还可以和一个类型绑定，使得其不但有类型属性，还有值属性，比如上面的例子，可以简化为这样：

```rust
#[derive(Debug)]
enum IpAddress {
    V4(String),
    V6(String)
}

fn main() {
    let ip = IpAddress::V4("127.0.0.1".to_string());

    println!("ip = {:#?}", ip);

}
```

这时的输出为：

```rust
ip = V4(
    "127.0.0.1",
)
```

可以看到，enum的类型还可以和其它类型绑定在一起，形成“类型+值”的关系，简化了代码。

enum绑定可以很灵活，可以绑定任何类型，只要你想绑定，就能绑定，比如：

```rust
#[derive(Debug)]
struct Person {
    name: String,
    age: u32
}

#[derive(Debug)]
enum IpAddress {
    V4(String),
    V6(String),
    RAW_V4(u8, u8, u8, String), // 绑定多个值
    VPerson(Person) // 绑定自定义struct
    Point{x: i32, y: i32},  // 绑定一个匿名struct
}
```

可以这样使用这个enum：

```rust
    let raw_ip = IpAddress::RAW_V4(127, 0, 0, 1, "this is a home address".to_string());
    let person_type = IpAddress::VPerson(Person { name: String::from("jack"), age: 100 });
    let point = IpAddress::Point { x: 100, y: 120 };
    println!("raw_ip = {:?}", raw_ip);
    println!("person_type = {:?}", person_type);
    println!("point = {:?}", point);
```

会打印：

```rust
raw_ip = RAW_V4(127, 0, 0, 1, "this is a home address")
person_type = VPerson(Person { name: "jack", age: 100 })
point = Point { x: 100, y: 120 }
```



### 1.2 给enum加上成员方法

enum甚至可以加上成员方法，比如我们给IpAddress添加一个方法：

```rust
impl IpAddress {
   fn do_something(&self) {
        println!("this is an enum!");
   }
}
```

此时，一旦拥有了一个IpAddress的枚举值，即可调用这个do_something方法：

```rust
    let ip = IpAddress::VPerson(Person { name: "jack".to_string(), age: 100 });
    ip.do_something();
```

将会打印：

```rust
this is an enum!
```



### 1.3 enum和struct的区别

看上去enum和struct大部分是一样的，它们可以定义字段，定义方法，但也有细微的不同，首先当然是一个是用struct关键字定义，一个是用enum定义。

其次，如果我们把上面的例子改成用struct实现，那么每一个枚举类型都是一个struct，那么这些struct它们都不属于一个类型下的定义，即每个类型都是独立的，而枚举类型则是打包了很多个类型，此时若把struct当作enum来用，if let判断或者match判断是错误的，比如：

```rust
struct IpAddressV4 {
    name: String
}

struct IpAddressV6 {
    age: u32
}

fn main() {
    let ip = IpAddressV6{
        age: 100
    }; 
    match ip {
        IpAddressV4 => println!("it is v4"),
        IpAddressV6 => println!("it is v6")
    }
}
```

上面的代码虽然可以编译，但打印出来的却是：

```rust
it is v4
```

很明显，不符合预期。 这是因为match发现它们都是struct这个类型，于是走进了V4这个分支中。用enum就可以顺利表达我们的意思：

```rust
enum IpAddress {
    V4(String),
    V6(u32)
}

fn main() {
    let ip = IpAddress::V6(100);
    match ip {
        IpAddress::V4(x) => println!("it is v4"),
        IpAddress::V6(x) => println!("it is v6"),
    }
}
```

此时打印：

```rust
it is v6
```

打印正常，因为enum能正确给V4和V6不同的类型，所以，如果需要用match或者if let的话，还是使用enum吧。



## 2. match控制流

上面的代码用到了match语法，match其实就是C++中的switch，关键是：match强制检查所有enum类型，否则报错，也就是必须要么枚举所有的类型，或者使用下划线 _  来指定匹配之前的类型都失败则执行默认行为。比如之前的match判断就枚举了V4和V6的所有枚举，但如果遇到整型，由于不可能枚举所有整形，就需要默认分支，也即下划线来保证所有的分支都有考虑到：

 ```rust
 fn is_24_or_30(i: i32) {
     match i {
         24 => println!("one day has 24 hours"),
         30 => println!("some monthes have 30 days"),
         _ => println!("no decriptions, it is {}", i),
     }
 }
 ```

注意，每一个rust分支都是以逗号“,”结尾。我们也可以加上大括号，写一些语句：

```rust
fn is_24_or_30(i: i32) {
    match i {
        24 => println!("one day has 24 hours"),
        30 => println!("some monthes have 30 days"),
        365 => {
            let p = Person {
                name: "jack".to_string(),
                age: 100,
            };
            println!("a year commonly has 365 days, the person is: {:?}", p);
        },
        _ => println!("no decriptions, it is {}", i),
    }
}
```



## 3. 一个常用的enum：Optional\<T\>

可以讲讲Optional\<T\>了。它实际上是一个enum，其源代码如下：

```rust
pub enum Option<T> {
    /// No value.
    #[lang = "None"]
    #[stable(feature = "rust1", since = "1.0.0")]
    None,
    /// Some value of type `T`.
    #[lang = "Some"]
    #[stable(feature = "rust1", since = "1.0.0")]
    Some(#[stable(feature = "rust1", since = "1.0.0")] T),
}
```

只有两个成员类型，一个是None，顾名思义，即空类型，一个是Some，绑定一个范型T，所以，如果一个变量是Optional\<T\>的，那么意味着，要么是None类型，要么是Some类型。例如我们写一个函数，入参是Optional\<T\>，那么里面就要判断两个类型，一个是Some \<T\>，另一个是None：

 ```rust
 #[derive(Debug)]
 enum IpAddress {
     V4(u8, u8, u8, u8),
     V6(String),
 }
 
 fn what_kind(k: &Option<IpAddress>) {
     match k {
        Some(x) => {
             match x {
                 IpAddress::V4(a, b, c, d) => println!("the v4 ip is {}:{}:{}:{}", a, b, c, d), 
                 IpAddress::V6(a) => println!("the v6 ip is {:?}", a), 
             }
        },
        None => println!("None"),
     }
 }
 
 fn main() {
     let ip1 = Some(IpAddress::V4(127, 0, 0, 1));
     let ip2 = Some(IpAddress::V6("::::1".to_string()));
     let ip3 = None;
 
     what_kind(&ip1);
     what_kind(&ip2);
     what_kind(&ip3);
 }
 ```

以上代码可以看出，遇到Optional\<T\>枚举时，必须考虑None和Some两种情况（不然编译报错，虽然可以用下划线来表示默认行为），而要想拿到和Some绑定的T，则必须用match（或者if let，下面会讲），简单总结来说，Optional\<T\>分为两层处理：

1、处理None和Some；

2、若不为None处理Some取出T。

那么，有人会问，为什么要有Optional\<T\>这个东西呢？因为Rust所有实体都不能为invalid状态，因此不存在“空”这样的概念，要么有值，要么不能访问，产生编译错误，但现实是有这个概念的，很多语言也有这个概念，Rust在语言机制层面禁止了这个状态的存在，可为了和现实或者和其它语言相对应表达意思的话怎么办呢？于是就需要Optional\<T\>来做这个工作。

最后要提一下的是，如果match某个枚举类型啥也不想干，怎么表达呢？因为match必须强制对所有的枚举类型都考虑一遍，所以，遇到啥也不想干的分支，需要一个特殊的写法：

```rust
    match ip1 {
        Some(x) => {
            match x {
                IpAddress::V4(a, b, c, d) => println!("here"),
                _ => (),
            }
            // println!("x = {:?}", x);
        }
        _ => (),
    }
```

上面代码第9行，=> ()就是表示，这个分支啥也不干。



## 4. if let用法

上面讲了match处理Optional\<T\>的方法，如果你只是对某个枚举类型感兴趣但其它都做默认处理的话，就可以使用if let用法。这样可以比较自然的写出对枚举类型的处理，其实也可以少打一些代码。

if let其实就是match的特殊版本，即只处理某一个枚举，其它要么忽视，要么else去做默认处理，例如上面的例子可以改写成这样：

```rust
fn what_kind(k: &Option<IpAddress>) {
    if let Some(x) = k {
        if let IpAddress::V4(a, b, c, d) = x {
            println!("the v4 ip is {}:{}:{}:{}", a, b, c, d);
        }
        if let IpAddress::V6(a) = x {
            println!("the v6 ip is {:?}", a);
        }
    } else {
        println!("None");
    }
}
```



## 5. 使用match和if let的注意事项

match和if let的使用算是讲完了，但有一个不起眼的事情需要提一下，就是match和if let是会发生资源转移的，例如：

```rust
    if let Some(x) = ip1 {
        if let IpAddress::V4(a, b, c, d) = x {
            println!("here");
        }
        println!("x = {:?}", x);
    }
    println!("ip1 = {:?}", ip1);
```

上面第一行代码if let Some(x) = ip1，会发生资源转移，即ip1的字段会转移到x上，也即partial moves（详见：https://doc.rust-lang.org/rust-by-example/scope/move/partial_move.html ）因此，第7行的println!会访问一个已经包含悬空指针的ip1，这不合法，编译器拒绝编译：

```rust
error[E0382]: borrow of partially moved value: `ip1`
  --> src/main.rs:71:28
   |
65 |     if let Some(x) = ip1 {
   |                 - value partially moved here
...
71 |     println!("ip1 = {:?}", ip1);
   |                            ^^^ value borrowed here after partial move
   |
   = note: partial move occurs because value has type `IpAddress`, which does not implement the `Copy` trait
   = note: this error originates in the macro `$crate::format_args_nl` (in Nightly builds, run with -Z macro-backtrace for more info)
help: borrow this field in the pattern to avoid moving `ip1.0`
   |
65 |     if let Some(ref x) = ip1 {
   |                 +++
```

编译起给出的建议是用引用ref x，当然，我觉得用&ip1也可以。

还需要注意到一个点是，x并没有发生资源转移，因为x的资源都是u8，他们是primitive类型的，直接使用拷贝，这在前面讲过。

换成match呢：

```rust
	 match ip1 {
        Some(x) => {
            if let IpAddress::V4(a, b, c, d) = x {
                println!("here");
            }
            println!("x = {:?}", x);
        },
        _ => (),
    }
    println!("ip1 = {:?}", ip1);
```

依然编译不通过的，资源转移发生在上面的第2行：Some(x) =>，此时ip1资源转给了x，ip1又包含悬空指针了：

```rust
error[E0382]: borrow of partially moved value: `ip1`
  --> src/main.rs:74:28
   |
66 |         Some(x) => {
   |              - value partially moved here
...
74 |     println!("ip1 = {:?}", ip1);
   |                            ^^^ value borrowed here after partial move
   |
   = note: partial move occurs because value has type `IpAddress`, which does not implement the `Copy` trait
   = note: this error originates in the macro `$crate::format_args_nl` (in Nightly builds, run with -Z macro-backtrace for more info)
help: borrow this field in the pattern to avoid moving `ip1.0`
   |
66 |         Some(ref x) => {
```

 总结来说，使用match或者if let一定要小心，如果不想发生资源转移，那么就用引用来代替，编译器给出了不错的建议。
