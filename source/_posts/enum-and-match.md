---
title: enum和match
date: 2023-01-08 19:25:24
published: false
tags:
categories:
- Rust
---

今天就来熟悉一下Rust的enum和match语法吧。

<!--more-->



## 定义一个enum

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



### 给enum绑定值

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



### 给enum加上成员方法

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



### enum和struct的区别

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



## match控制流

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



## 一个常用的enum：Optional\<T\>

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





## if let用法

if let实际上





## 
