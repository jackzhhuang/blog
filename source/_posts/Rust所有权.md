---
title: Rust所有权
date: 2023-01-02 16:17:40
tags:
- Rust
categories:
- Rust
---

Rust的所有权应该是Rust语言最难理解的一个，是初学者学习Rust遇到的第一个也是最大的陡坡。很多语言几乎不需要去花时间学习资源管理（Java，Go，Python等，C++当然是除外的），所以那些语言很流行，很受欢迎，因为只需要学习一下语法，关键字就能投入到项目中去了，但Rust学习单单是所有权（相当于资源管理）这一块内容，就可以成功劝退绝大部分程序员。

这是Rust的劣势，决定了Rust流行不起来，也是Rust的优势，决定了Rust效率和正确率上都远优于其它语言（甚至包括C语言）。今天写个总结，把Rust所有权内容说一说。

<!--more-->



## 1. 简单的生命周期

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



## 2. 从最简单的所有权关系说起

### 2.1 赋值

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



### 2.2 参数传递

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



## 3. 使用借用

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



## 4. 使用克隆

另一个可以用于赋值的方法，就是克隆函数clone：

```rust
    let s = String::from("hello rust! ");
    let t = s.clone();
    println!("t = {}, t = {:p}", t, &t);
    println!("s = {}, s = {:p}", s, &s);
```

这时，s和t就是两个完全不同的指针了，它们只是值相同而已：

![克隆](https://www.jackhuang.cc/svg/rust_clone.svg)



## 5. 数组的特殊情况

以上说的是非数组情况，到了数组，稍微有点不一样了，因为数组是维护了一组指针，如果随便允许把其中某个或者某些指针置为悬空指针，那么数组就需要维护哪个指针是悬空的，哪个指针是有效的。Rust拒绝这么干，所以，数组中的指针不允许直接变成悬空指针：

```rust
    let array: [String; 5] = [String::from("hello"), 
                              String::from("rust"), 
                              String::from("!"), 
                              String::from("jack"), 
                              String::from("huang"),];
    let t = array[2];
    let s = array[3];

    println!("t = {}", t);
    println!("s = {}", s);

```

按前面所说，以上代码中，t和s会获得array数组中第三、第四个资源所有权，但实际Rust拒绝这么干，此时编译错误：

```rust
error[E0508]: cannot move out of type `[String; 5]`, a non-copy array
  --> src/main.rs:21:13
   |
21 |     let t = array[2];
   |             ^^^^^^^^
   |             |
   |             cannot move out of here
   |             move occurs because `array[_]` has type `String`, which does not implement the `Copy` trait
   |             help: consider borrowing here: `&array[2]`

error[E0508]: cannot move out of type `[String; 5]`, a non-copy array
  --> src/main.rs:22:13
   |
22 |     let s = array[3];
   |             ^^^^^^^^
   |             |
   |             cannot move out of here
   |             move occurs because `array[_]` has type `String`, which does not implement the `Copy` trait
   |             help: consider borrowing here: `&array[3]`
```

也即，对于数组中的资源，我们只能用引用访问，以下可以正常编译通过且打印如预期：

```rust
    let array: [String; 5] = [String::from("hello"), 
                              String::from("rust"), 
                              String::from("!"), 
                              String::from("jack"), 
                              String::from("huang"),];
    let t = &array[2];
    let s = &array[3];

    println!("t = {}", *t);
    println!("s = {}", *s);
```

相比之前的代码，t和s都变成了引用，由于是引用，并不会转移所有权，因此array的资源都在。

如果是多个引用，则直接使用引用数组即可，例如连续引用数组的第三和第四个资源：

```rust
    let array  = vec![String::from("hello"), 
                              String::from("rust"), 
                              String::from("!"), 
                              String::from("jack"), 
                              String::from("huang"),];
    let t: &[String] = &array[2..=3];

    println!("{:?} ", t);
```

此时打印：

```rust
["!", "jack"] 
```

这里，数组被我改成了Vec对象，其实只要是连续的对象，比如数组（即[]），Vec，String都可以用引用数组来引用，String的引用有点特殊：

```rust
    let s = String::from("hello rust!");
    let t: &str = &s[2..=6];

    println!("{:?} ", t);
```

这里看到，使用的是&str而不是&[T]，这是Rust的两类数组，&[T]是普通数组引用，&str相当于对字符类型的特殊数组引用，即char串引用。作为特殊的引用，Rust还给它们（&[T]和&str）起了个名字叫切片，即slice。关于引用后面还要专门拿出来讲，这里关注所有权这个话题就好了。

尽管数组不允许别人把它的资源拿走，只能用引用，但如果用迭代器去访问，资源的所有权还是会被拿走的：

```rust
    let v = vec![String::from("hello"), 
                              String::from("rust"), 
                              String::from("!"), ];
    for t in v {
        print!("t = {}", t)
    }                              

    println!("v = {:?}", v);
```

上面的代码会报错，因为v的资源的所有权都被t拿走了，v已经是一个悬空指针：

```rust
  --> src/main.rs:32:26
   |
25 |     let v = vec![String::from("hello"), 
   |         - move occurs because `v` has type `Vec<String>`, which does not implement the `Copy` trait
...
28 |     for t in v {
   |              - `v` moved due to this implicit call to `.into_iter()`
...
32 |     println!("v = {:?}", v);
   |                          ^ value borrowed here after move
```



## 6. primitive类型不会有所有权转移

尽管前面讲了很多所有权转移的例子，但对于原始类型（比如整型，bool，char）并不会有所有权转移的问题，它们永远都是copy：

```rust
    let a = 3;
    let b = a; // 不会发生所有权转移，a和b都有自己的值3
    println!("a = {}, b = {}", a, b);
```

上面的代码正常编译和运行：

```rust
a = 3, b = 3
```



所有权和引用后面还会专门讲很多，这次算是先来个前奏。慢慢积累吧。
