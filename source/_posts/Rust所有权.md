---
title: Rust所有权
date: 2023-01-02 16:17:40
published: false
tags:
categories:
- Rust
---

Rust的所有权应该是Rust语言最难理解的一个，是初学者学习Rust遇到的第一个也是最大的陡坡。很多语言几乎不需要去花时间学习资源管理（Java，Go，Python等，C++当然是除外的），所以那些语言很流行，很受欢迎，因为只需要学习一下语法，关键字就能投入到项目中去了，但Rust学习单单是所有权（相当于资源管理）这一块内容，就可以成功劝退绝大部分程序员。

这是Rust的劣势，决定了Rust流行不起来，也是Rust的优势，决定了Rust效率和正确率上都远优于其它语言（甚至包括C语言）。今天写个总结，把Rust所有权内容说一说。



## 简单的生命周期

从最简单的说起，let表示绑定某个指针到某个对象上，当离开作用域时，对象被销毁，指针进而变为invalid：

```rust
fn main() {
    let s = String::from("hello rust!");
}
```

例如上面的代码，离开main函数，就变成invalid了，String对象也销毁了。

上面的代码相当于这样：

![绑定关系](https://www.jackhuang.cc/svg/rust_所有权.svg)

这里简单说一下，s可以理解为一个指针，String即为对象，其成员p指向真正的字符串类型，后面的10位字符串的长度，16为String对象的容量（capacity） 。

## 从最简单的所有权关系说起

### 赋值

其实Rust难，就难在下面这个简单代码里面，很多人是无法接受Rust这种规定，但其实Rust这么干也是用心良苦。

```rust
    let s = String::from("hello rust!");
    let t = s;
    println!("t = {}", t);
    println!("s = {}", s);
```

以上代码是不能通过编译的，报错如下：

```rust
  --> src/main.rs:17:24
   |
14 |     let s = String::from("hello rust!");
   |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
15 |     let t = s;
   |             - value moved here
16 |     println!("t = {}", t);
17 |     println!("s = {}", s);
   |                        ^ value borrowed here after move
```

因为s赋值给t后，s将会变成invalid，也就是悬空指针，这个时候不能对s进行任何访问操作，其对象关系图如下：

![赋值](https://www.jackhuang.cc/svg/rust赋值.svg)

也即，s将失去String对象，t取而代之，只有去掉对s的访问才可以通过编译，Rust不允许访问没有资源的指针。



### 参数传递

同理，返回值也一样，比如我们有一个函数，返回一个String对象：

```rust
fn print_upper_string (t: String) {
    println!("{}", t.to_uppercase());
}


fn main() {
    let s = String::from("hello rust!");
    print_upper_string(s);
    println!("s = {}", s);
}
```

和之前一样，上面的代码第10行诗没有办法通过编译的，因为print_upper_string函数的参数t已经获得了s的String对象资源，s变成了悬空指针，cargo build会提示：

```rust
error[E0382]: borrow of moved value: `s`
  --> src/main.rs:16:24
   |
14 |     let s = String::from("hello rust!");
   |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
15 |     print_upper_string(s);
   |                        - value moved here
16 |     println!("s = {}", s);
   |                        ^ value borrowed here after move
```



## 使用借用

解决以上的问题有三个办法，一个是借用，一个是clone，还有一个是使用引用计数。这里先讲借用。

所谓借用，其实就是C++的引用，也就是被赋值的指针是引用，而不是拥有资源，此时不发生资源转移：

```rust
    let s = String::from("hello rust!");
    let t = &s;
    println!("t = {}", t);
    println!("s = {}", s);
```

例如上面的代码，t获得&s，即t只是借用s资源，并不拥有它，因此可以通过编译，运行良好。都会打印hello rust!：

```rust
t = hello rust! 
s = hello rust! 
```

如果去打印t的值和s的地址，会发现它们是相等的：

```rust
    println!("t = {:p}", t);
    println!("s = {:p}", &s);
```

此时会输出：

```rust
t = 0x7ffeefbff2f8
s = 0x7ffeefbff2f8
```

所以t其实对s的一个引用：

![引用](https://www.jackhuang.cc/svg/rust引用.svg)

因此，打印t和s所指向的字符串内容时，访问t也可以用解引用来访问：

```rust
    println!("t = {}", *t);
    println!("s = {}", s);
```

以上两句代码同样也是打印hello rust!。而且，你会发现*t的地址正是s的地址。

## 使用克隆

另一个可以用于赋值的方法，就是克隆函数clone：

```rust
    let s = String::from("hello rust! ");
    let t = s.clone();
    println!("t = {}, t = {:p}", t, &t);
    println!("s = {}, s = {:p}", s, &s);
```

这时，s和t就是两个完全不同的指针了，它们只是值相同而已：





## 数组的特殊情况



## 使用RC计数器



