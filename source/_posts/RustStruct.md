---
title: Rust的struct类型
date: 2023-01-07 17:19:26
toc: true
tags:
- Rust
categories:
- Rust
---

Rust的struct定义和C++的类似，甚至提供了简化初始化的方法，定义成员方法也更灵活，刚刚登上第一个陡坡（所有权），这一节缓缓，因为struct算是Rust中难度	不高的内容。

<!--more-->

<!-- toc -->

## 1. struct的定义

和很多语言一样，定义一个struct就是用struct关键字加上名字，然后罗列出字段来，比如最简单的定义一个人：

```rust
struct Person {
    name: String,
    email: String, 
    age: u32,
    address: String
}
```

这就定义好了一个人，接下来就是初始化了。



## 2. struct的初始化方法

### 2.1 直接初始化

Rust有三种办法初始化一个struct，最简单的莫过于直接赋值，很像json，即key-value形式赋值即可（另外其实提示一下，初始化这些字段并无顺序要求）：

```rust
    let one = Person {
        name: String::from("Jack"),
        email: String::from("jackzhhuang@gmail.com"),
        age: 100,
        address: "China".to_string()
    };
```

试着打印one，会发现错误：

```rust
    println!("one = {}", one);
```

错误信息为

```rust
error[E0277]: `Person` doesn't implement `std::fmt::Display`
  --> src/main.rs:34:26
   |
34 |     println!("one = {}", one);
   |                          ^^^ `Person` cannot be formatted with the default formatter
   |
   = help: the trait `std::fmt::Display` is not implemented for `Person`
   = note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead
   = note: this error originates in the macro `$crate::format_args_nl` (in Nightly builds, run with -Z macro-backtrace for more info)
```

这里根据提示，会知道，Rust并不是说不知道one里面有什么东西，而是它不知道怎么格式化输出这个对象，好在我们有Debug trait，它可以告诉Rust怎么格式化输出结构，所以，Person的定义加上Debug trait即可：

```rust
#[derive(Debug)]
struct Person {
    name: String,
    email: String, 
    age: u32,
    address: String
}
```

Rust提供了很多trait，struct结构derive了某个trait就有了某个特性，比如这里derive了Debug trait，就可以让rust知道怎么格式化输出这个结构体，更多trait后面会讲到，官方教学文档也给所有的trait介绍，详见：https://doc.rust-lang.org/book/appendix-03-derivable-traits.html，这里不展开说。

另外，由于是自定义的类型，需要在格式化的时候告诉Rust，因此print宏中，{}需要修改为{:?}（简洁打印，所有字段都在一行打印出来）或者{:#?}（美观打印，会换行且对齐）。

此时，可以正常打印one了，以下是美观打印出来的输出：

```rust
one = Person {
    name: "Jack",
    email: "jackzhhuang@gmail.com",
    age: 100,
    address: "China",
}
```



### 2.2 用函数初始化

我们当然可以写一个函数然后传入一些字段值，然后在函数中初始化一个对象，最后返回出去。但Rust还提供了一个更简略的语法：

```rust
fn build_a_person(name: String, age: u32) -> Person {
    Person { name, 
             age, 
             address: "China".to_string(), 
             email: "jackzhhuang@gmail.com".to_string()
    }
}
```

可以看到，name和age直接就放在大括号中了，并不需要写出name: name或者age: age这样的代码。



### 2.3 用别的对象初始化

如果已经有了一个对象，我们还想初始化另一个对象，只是有些字段和现有的对象字段不同，除了用前面的方法初始化以外，还有什么办法吗？Rust提供了struct update语法：

 ```rust
     let one = Person {
         name: String::from("Jack"),
         email: String::from("jackzhhuang@gmail.com"),
         age: 100,
         address: "China".to_string()
     };
 
 
     let another = Person {
         name: "Bob".to_string(),
         email: "jackhuangvvv@gmail.com".to_string(),
         ..one
     };
 
     println!("another = {:#?}", another);
 ```

以上会输出：

```rust
another = Person {
    name: "Bob",
    email: "jackhuangvvv@gmail.com",
    age: 100,
    address: "China",
}
```

可以看到，age和address都和one一样有同样的值。

但此时注意，one的address变成了悬空指针，此时不能再访问，如果后面加上访问代码，会直接编译错误：

```rust
    println!("one.name = {}", one.name);    // 没问题
    println!("one.email = {}", one.email);  // 没问题
    println!("one.age = {}", one.age);      // 没问题，因为primitive类型是copy
    println!("one.address = {}", one.address);   // 编译错误，因为address已经被转移到another了
```

编译结果为：

```rust
44 |       let another = Person {
   |  ___________________-
45 | |         name: "Bob".to_string(),
46 | |         email: "jackhuangvvv@gmail.com".to_string(),
47 | |         ..one
48 | |     };
   | |_____- value moved here
...
55 |       println!("one.address = {}", one.address);
   |                                    ^^^^^^^^^^^ value borrowed here after move
```

即这么干，one会把它的字段资源转移到another，由于name和email我们赋的是新值，而age是u32，为primitive，赋值是copy形式，只有address字段我们即没有给新值，且其赋值会采取资源转移形式，因此one的address变成悬空指针了，不能访问。



## 3. 特殊的struct

和C++一样，可以定义一个什么都没有的struct：

```rust
#[derive(Debug)]
struct Nothing;

fn main() {
    let no = Nothing;
    println!("n = {:?}", no)
}
```

 这个用在trait比较多，类似于定义一个interface，后续会学习到。



## 4. 给struct加上方法

在没有成员方法之前，我们计算一个长方形的面积时这么写的：

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn area(width: u32, height: u32) -> u32 {
    width * height
}

fn main() {
    let r = Rectangle {
        width: 60,
        height: 80,
    };

    println!("the area of the rectangle = {:?} is {}", r, area(r.width, r.height));
}

```

用面向对象的方法去思考，面积是长方形的一个属性，长方形自己才知道如何计算自己的面积，因此，应该是长方形这个类返回面积才对，把area这个函数变成Rectangle的方法，只需要impl关键字，并且把area移到Rectangle里面去，并且，第一个参数是&self：

```rust
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }   
}
```

此时，打印时只需要简单的点area调用即可：

```rust
println!("the area of the rectangle = {:?} is {}", r, r.area());
```

这里需要注意的一个地方是，self这个参数，我们用了引用，其实是省略写法，写完整是self: &Self，那么如果不使用引用可以吗？当然可以，只是这么干，会触发Rust的资源转移机制，调用方的对象会变悬空指针：

```rust
impl Rectangle {
    fn area(self) -> u32 {
        self.width * self.height
    }   
}

let s = r.area();
println!("the area of the rectangle = {:?} is {}", r, s);
```

因为self不是引用，上面的代码第7行r会变成悬空指针，那么第8行的打印r会编译不通过：

```rust
20 |     let s = r.area();
   |               ------ `r` moved due to this method call
21 |     println!("the area of the rectangle = {:?} is {}", r, s);
   |                                                        ^ value borrowed here after move
   |
note: this function takes ownership of the receiver `self`, which moves `r`
  --> src/main.rs:9:13
   |
9  |     fn area(self) -> u32 {
   |             ^^^^
   = note: this error originates in the macro `$crate::format_args_nl` (in Nightly builds, run with -Z macro-backtrace for more info)
```

根据提示可以知道，因为self不是引用，所以对象资源被转移走了，r就丢了对象，导致悬空指针。



## 5. 静态方法

正如之前用的String::from那样，struct可以定义静态成员方法，也就是不需要对象指针就可以访问的方法，只需要在方法上不加入self参数即可，比如下面这个计算一个数的平方：

```rust
impl Rectangle {
    fn square(n: u32) -> u32 {
        n * n
    }
}

let n = 3;
println!("square of {} is {}", n, Rectangle::square(n));
```

可以正常打印：

```rust
square of 3 is 9
```



## 6. 定义多个成员方法

定义多个成员方法时，可以都写在同一个impl块中也可以分开写：

```rust
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }   

    fn square(n: u32) -> u32 {
        n * n
    }
}
```

和以下是等价的：

```rust
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }   

}

impl Rectangle {
    fn square(n: u32) -> u32 {
        n * n
    }
}
```

可以分开写在不同的impl块的好处是扩展Rectangle类的时候不需要去修改之前的老代码，相比OOP中的写一个子类去扩展要更方便更灵活。



## 7. 遗留问题

这一节相对来说是比较简单的，实际上遗留了一些比较难的问题，这些需要后面学习到lifetime的时候解决：

1、目前为止我们都是用的类来做成员字段，如果用引用做成员字段呢？

2、如果struct对象已引用的方式传参，那么会发生什么？

3、如何解决用其它对象初始化新对象时，其它对象字段指针变成悬空指针的问题？
