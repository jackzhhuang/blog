---
title: Rust的box容器
date: 2023-01-28 08:22:51
tags:
- Rust
- Box<T>
categories:
- Rust
---

Box\<T\>就是模仿Rust的指针行为：

1、可以解引用访问，即*a操作；

2、超出生命范围区域会主动释放资源。

Rust还提供了Rc\<T\>，RefCell\<T\>等专门为各种场景使用的容器类，内容太多太细节，打算都当作单独的一节来讲，这一节就专注于box\<T\>。

 <!--more-->



## 为什么需要 Box\<T\>

Rust最大的优点定（当然可能也是一把双刃剑）就是编译的时候确定实体大小。但总有需要运行时才知道的时候，例如多态（往后会讲），或者例如本例这种struct/enum类型：

```rust
enum List {
    Con(i32, List),
    Nil,
}
```

这个 List 枚举的Con 类型绑定了一个 i32 和 另一个 List，因此它的大小在编译的时候无论如何都是未知的。编译失败：

```rust
 --> src/main.rs:2:1
  |
2 | enum List {
  | ^^^^^^^^^ recursive type has infinite size
3 |     Con(i32, List),
  |              ---- recursive without indirection
```

这是因为编译的时候编译器就需要给它一个大小定义。这时就可以使用Box\<T\>来做到确定的大小。当然Box\<T\>不是未卜先知，而是编译器看到Box\<T\>就知道它是一个固定大小的struct，因为Box\<T\>封装了T的原始指针，它的大小是已知的。我们修改List定义，并sizeof打印它的大小：

```rust
enum List {
    Con(i32, Box<List>),
    Nil,
}

fn main() {
    println!("sizeof i32 = {}", std::mem::size_of::<i32>());
    println!("sizeof Box<List> = {}", std::mem::size_of::<Box<List>>());
    println!("sizeof Box<f64> = {}", std::mem::size_of::<Box<f64>>());
    println!("sizeof Box<i16> = {}", std::mem::size_of::<Box<i16>>());
    println!("sizeof list = {}", std::mem::size_of::<List>());
}
```

输出如下：

```rust
sizeof i32 = 4
sizeof Box<List> = 8
sizeof Box<f64> = 8
sizeof Box<i16> = 8
sizeof list = 16
```

这里可以看到，不管 Box\<T\> 的 T 是什么，其大小都是 8，因为  Box\<T\>其实就是一个封装了 unsafe 原始指针，不管 T 是什么，其大小都是 *mut T，即 8个字节（指针大小）：

```rust
struct Box<T> {
    ptr: *mut T,
}
```

Rust的标准库大多数都是使用unsafe机制实现的，也就是unsafe代码实现了一套safe库。在其中用sta::alloc库实现了 unsafe 内存分配和释放。



## 实现一个自己的 Box\<T\> —— 理解 Deref 和 Drop trait

### 定义 MyBox\<T\>

MyBox\<T\>是一个空struct，但它绑定了一个 T 对象：

```rust
struct MyBox<T>(T);
```

new的时候返回MyBox\<T\>对象即可：

```rust
impl<T> MyBox<T> {
   pub fn new(x: T) -> MyBox<T> {
        MyBox(x)
   } 
}
```



### deref 方法

为了让MyBox\<T\>能做到解引用（*），我们必须实现 Deref trait，其中就是要实现deref方法，它就是返回一个 &T：

```rust
impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```



### drop 方法

接着就是Drop trait，即在超出作用域的时候会调用，类似析构函数：

```rust
impl<T> Drop for MyBox<T> {
    fn drop(&mut self) {
        println!("nothing to do but println it to see it was called");
    }
}
```



### 观察分配和析构

```rust
fn main() {
    let a = MyBox(100);
    println!("what the a is: {}", *a);
}
```

这里 a 将会 act like a dynamic pointer，即支持 *a 运算（相当于调用了 a.deref() ），当作用域走出 main 函数后，a会自动销毁，此时a.drop() 被调用：

```rust
what the a is: 100
nothing to do but println it to see it was called
```



### 主动析构

有些时候我们想主动析构一个box对象，比如主动释放连接之类的，但如果我们主动的调用drop是会编译失败的：

```rust
fn main() {
    let a = Box::new(100);
    println!("what the a is: {}", *a);
    a.drop();
}
```

编译会发现编译失败：

```rust
  --> src/main.rs:23:7
   |
23 |     a.drop();
   |     --^^^^--
   |     | |
   |     | explicit destructor calls not allowed
   |     help: consider using `drop` function: `drop(a)`
```

这里也给出了提示，使用std::mem::drop 函数：

```rust
fn main() {
    let a = Box::new(100);
    println!("what the a is: {}", *a);
    std::mem::drop(a);
}
```



## Box\<T\> 的非基本操作

Box\<T\> 提供了一些方法供我们访问被封装的对象，但注意这种方式打破了 Box\<T\>的封装，不过好在 Box\<T\> 是编译时就能检查所有权和引用规则，所以一般不会出问题（要出也是编译时期就被检查出来），相对这个来讲，RefCell 就比较危险了。



### 使用*解引用

解引用操作会使 Box\<T\> 返回被封装的对象，这会造成 Box\<T\>  失去对 T 对象的所有权：

```rust
   let mut x = Box::new(Person {
        age:99,
    });
    let mut y = *x;
    println!("x = {:?}, y = {:?}", x, y);
```

上面的第 4 行代码会使得 x 失去对 Person对象的所有权，编译失败：

```rust
  --> src/main.rs:16:36
   |
15 |     let mut y = *x;
   |                 -- value moved here
16 |     println!("x = {:?}, y = {:?}", x, y);
   |                                    ^ value borrowed here after move
```

当然其实 改为 let mut y = x; 也会使 x:  Box\<T\>  失去所有权。



### as_ref

使用as_ref 方法则获得 Box\<T\> 的 T 引用。除非函数参数必须传入引用，否则一般情况下也不需要使用这个方法：

```rust
#[derive(Debug)]
struct Person {
    age: i32,
}

fn print_age(p: &Person) {
    println!("the age of p is {}", p.age);
}

fn main() {
    let x = Box::new(Person {
        age:99,
    });
    print_age(x.as_ref());
    println!("the age of x is {}", x.age);
}
```

因为是返回的引用，不像解引用那样，x 不会失去所有权。



### as_mut

相比as_ref，as_mut 则返回一个可变引用：

```rust
#[derive(Debug)]
struct Person {
    age: i32,
}

fn change_age(p: &mut Person) {
    p.age = 100;
    println!("the age of p is {}", p.age);
}

fn main() {
    let mut x = Box::new(Person {
        age:99,
    });
    change_age(x.as_mut());
    println!("the age of x is {}", x.age);
}
```

同样，因为是返回的引用，不像解引用那样，x 不会失去所有权。



## 什么时候需要使用 Box\<T\>

总结看来，Box\<T\> 是一个其实是对 unsafe 指针的封装，那么什么时候会使用Box\<T\> 呢？一般以下几种情况可以使用：

1、当 T 是一个大 size 类型时，可以减少拷贝；

2、需要实现堆的内存特性，即如果资源超出作用域依然不想析构，此时可以用 Box\<T\> 传递出去；例如在函数中分配内存，在函数外使用，如果不使用 Box\<T\>，那么代码看起来像这样： 

```rust
fn school_maker(t: SchoolType) -> dyn SelfIntroduce {
    match t {
        SchoolType::school_student => Student {
            name: String::from("jack"),
            no: 47,
        },
        SchoolType::school_teacher => Teacher {
        name: String::from("bill"),
        id: 99265247,
        },   
    }
}
```

Student 和 Teacher 两个对象都会在函数结束的时候被析构调，外面的使用方将会变成悬挂指针。但如果改成Box\<T\>：

```rust
fn school_maker(t: SchoolType) -> Box<dyn SelfIntroduce> {
    match t {
        SchoolType::school_student => Box::new(Student {
            name: String::from("jack"),
            no: 47,
        }),
        SchoolType::school_teacher => Box::new(Teacher {
        name: String::from("bill"),
        id: 99265247,
        }),   
    }
}
```

 当函数返回的时候，内部的 Box\<T\> 将指针所有权全传递给外面使用方，指针所指资源依然存在不会被析构。



3、正如上面的例子，Student 和 Teacher 都实现了 SelfIntroduce trait，因此可以使用 Box\<T\> 来实现多态；

4、也正如最开始的例子，如果是一个无限大的 struct/enum，那么也可以用 Box\<T\>，因为 Box\<T\> 是对指针的封装，大小总是固定的，可以在编译时确定。



## Box\<T\> 要留意的地方

1、Box\<T\> 只能有一个所有权方；

2、Box\<T\> 只能单线程使用。

那么如果需要多个所有权就需要后面讲的 Rc\<T\> 。

