---
title: Rust 的高级特性
date: 2023-02-03 18:57:38
tags:
categories:
- Rust
---

今天讲一下 Rust 的高级特性。

包括：

1、unsafe 代码；

2、高级trait；

3、高级类型；

4、高级函数和闭包；

5、宏。

<!--more-->

## unsafe 代码

### 使用 raw 指针

最 unsafe 的行为莫过于直接使用 raw 指针，这时会绕过 Rust 的安全检查，比如经典的不能同时有可变和不可变引用规则，若使用 raw 指针，那么就需要我们很清楚的说，这是 unsafe 代码，Rust 才放心的运行：

```rust
   let mut a = 5;

    let x = &a as *const i32;
    let y = &mut a as *mut i32;

    unsafe {
        println!("*x = {}, *y = {}", *x, *y);
    }
```

类似的还有比如存在两个&mut 引用同一个变量：

```rust
fn split_at_mut(a: &mut [i32], index: usize) -> (&mut [i32], &mut [i32]) {
    assert!(a.len() >= index);
    (&mut a[..index], &mut a[index..])
}

fn main() {
    let mut a = [1, 2, 3, 4, 5];
    println!("split at 3: {:?}", split_at_mut(&mut a, 3));
}
```

上面的代码中，split_at_mut 函数最后一行两次 &mut a 会违反引用规则：

```rust
2 | fn split_at_mut(a: &mut [i32], index: usize) -> (&mut [i32], &mut [i32]) {
  |                    - let's call the lifetime of this reference `'1`
3 |     assert!(a.len() >= index);
4 |     (&mut a[..index], &mut a[index..])
  |     -----------------------^----------
  |     |     |                |
  |     |     |                second mutable borrow occurs here
  |     |     first mutable borrow occurs here
  |     returning this value requires that `*a` is borrowed for `'1`
```

这是因为 Rust 担心两个引用数组所包含的区间会相互覆盖。如果我们告诉 Rust 这个是 unsafe 代码，那么， Rust就会让编译通过并运行：

```rust
fn split_at_mut(a: &mut [i32], index: usize) -> (&mut [i32], &mut [i32]) {
    assert!(a.len() >= index);
    let p = a.as_mut_ptr();
    unsafe {
        (slice::from_raw_parts_mut(p, index),
         slice::from_raw_parts_mut(p.add(index),  a.len() - index))
    }
}
```



### 调用 unsafe 函数或者方法

如果一个函数或者方法是 unsafe 的，那么当然我们也必须告诉 Rust 我们知晓这个行为 unsafe，运行吧。



### 调用 C 函数也是 unsafe 的

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```

上面的代码中，调用 C 函数的 abs，extern "C" 表示调用 C 函数库。

当然反过来， C 函数调用 Rust 函数却是 safe 的，这里不展开来说。关于 C 和 Rust 互调以后有机会再说。



### 访问静态可变变量也是 unsafe 的

Rust 中，如果按照存储特性来分，有三种类型：

const：运行时存放于常量区，非运行时被固化在文件中；const 数据允许出现在任何地方定义。

static：进程开始后有固定的内存地址；static 数据允许出现在任何地方定义。

堆栈：堆栈数据则动态分配。堆栈数据则只能在局部中定义。

这里，若 static 是可变的，那意味着很多地方都可以去访问 static mut，这当然相当于打破了一个变量值允许一个 mut 引用的规则：

```rust
fn main() {
    static mut a: i32 = 9;
    a += 1;
}
```

  此时编译输出：

```rust
 --> src/main.rs:5:5
  |
5 |     a += 1;
  |     ^^^^^^ use of mutable static
```

明确告诉我们，去使用一个 static mut 的变量是不允许的，应该用unsafe：

```rust

fn main() {
    static mut a: i32 = 9;

    unsafe {
        a += 1;
    }
}
```



### 实现 unsafe trait 也当然是 unsafe 代码

 这个调用了 unsafe 函数或者方法一样。



### 使用union 也是 unsafe 的

和 C 一样，Rust 也有union，它是 unsafe的。



### 什么时候用 unsafe

自己能保证没问题的时候，例如上面的 split 数组，很明显，不管怎么样，split 出来的数组都不会相互有空间的覆盖，虽然 Rust 有这方面的考虑，但既然我们能确定不会有，那么我们就告诉 Rust，这个是已知 unsafe 代码，已确保无误。



## 高级 trait

### type

type 是 trait 的其中特性之一，比如我们有一个 trait 用于表示给人加年龄：

```rust
trait AddAge {
    type item;
    fn add(&mut self) -> Self::item;
}
```

 其中的 item 即某种类型，实现这个 trait 的 struct 来决定它是什么类型：

```rust
truct Person {
    age: i32,
    name: String,
}

impl AddAge for Person {
   type item = i32;

    fn add(&mut self) -> Self::item {
        self.age += 1;
        self.age
    } 
}
```

那么，为什么不用泛型呢？它和泛型区别在于，泛型可以定义多个 Person 实例，而 type 只能一个，比如我们将 AddAge 修改成使用泛型：

```rust
trait AddAge<T> {
    fn add(&mut self) -> T;
}

struct Person {
    age: i32,
    name: String,
}

impl AddAge<i32> for Person {
    fn add(&mut self) -> i32 {
        println!("in i32");
        self.age += 1;
        self.age
    } 
}

impl AddAge<u32> for Person {
    fn add(&mut self) -> u32 {
        println!("in u32");
        self.age += 1;
        self.age as u32
    } 
}
```

可以看到，我们可以定义多个 Person::add，根据不同的类型调用不同的 add：

```rust
fn main() {
    let mut a = Person {
        age: 99,
        name: String::from("jack"),
    };

    let x: i32 = a.add();
    let y: u32 = a.add();
}
```

输出为：

```rust
in i32
in u32
```

而使用 type 方法，我们就只能定义一种 add 方法了。

究其原因，还是因为泛型是类型的一部分，所以两个 add 是两个类型，即两个不同的 add 方法。但 type 则只能选其一。



### 默认类型参数

有时我们可以给 trait 的泛型加上默认类型参数，比如上面的例子中， AddAge 这个 trait 大概率就是给 Person 增加年龄，那么，我们可以默认的写出一些代码，使用方只需关心哪个字段是 age 就行：

```rust
trait AddAge<T = Self> {
    type item;
    fn add(&mut self) -> Self::item;
}

struct Person {
    age: i32,
    name: String,
}

impl AddAge for Person {
    type item = i32;
    fn add(&mut self) -> i32 {
        self.age += 1;
        self.age
    } 
}
```

 上面的 T = Self 即实现 AddAge 的类型。即默认情况下，使用实现这个 trait 的类型。



### 消除相同方法歧义

如果一个 struct 实现了两个 trait，而这两个 trait 有相同的方法怎么办呢？此时就需要显式的指定调用的是哪个 trait 的方法：

```rust
trait AddAge<T = Self> {
    type item;
    fn add(&mut self) -> Self::item;
}

trait AddFamilyName<T = Self> {
    type item;
    fn add(&mut self) -> Self::item;
}

struct Person {
    age: i32,
    name: String,
}

impl AddAge for Person {
    type item = i32;
    fn add(&mut self) -> i32 {
        self.age += 1;
        self.age
    } 
}

impl AddFamilyName for Person {
    type item = String;

    fn add(&mut self) -> Self::item {
        self.name = String::from("Huang ") + &self.name;
        self.name.clone()
    }
}

fn main() {
    let mut a = Person {
        age: 99,
        name: String::from("jack"),
    };

    println!("add age: {}", AddAge::add(&mut a));
    println!("add family name: {}", AddFamilyName::add(&mut a));
}
```

可以看到，AddAge 和 AddFamilyName 都有一个叫 fn add(&mut self) -> Self::item; 的方法，那么，在调用它们的时候，需要显式指定是哪一个的 add 方法，如上面的 39 和 40 行所显示。

但如果是静态 trait 方法呢？比如这个：

```rust
trait AddAll {
    type item;
    fn add() -> item;
}

impl AddAll for Person {
    type item = i32;

    fn add() -> Self::item {
        println!("do something...");
        0
    }
}
```

 此时需要使用模板的表达告诉编译器使用 AddAll的 add 方法：

```rust
println!("add all: {}", <Person as AddAll>::add());
```



### trait 还可以增加 bond

可以给 trait 增加 bond 限制实现实体必须满足其它的 trait 才可以实现：

```rust
trait OutlinePrint: fmt::Display {
    fn print(&self) {
        println!("from outline print: {}", self.to_string());
    }
}

struct Person {
    age: i32,
    name: String,
}

impl OutlinePrint for Person {
    
}
```

以上代码，第一行的trait 定义中， OutlinePrint 需要实现实体必须能满足 fmt::Display 输出的trait，但 Person 没有，此时 impl OutlinePrint for Person  会报错：

```rust
  --> src/main.rs:19:21
   |
19 | trait OutlinePrint: fmt::Display {
   |                     ^^^^^^^^^^^^ required by this bound in `OutlinePrint`

```

加上 Display 的实现后（可以用 to_string 方法了）正常编译和运行：

```rust
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "i am {} years old and my name is {}", self.age, self.name);
        Ok(())
    }
}
```



###  给外部 struct 增加 trait

如果我们想给不受我们控制的 struct 增加 trait 怎么办？简单的方法当然是用一个新的 struct 把它包裹起来，然后用新 struct 增加新 trait：

```rust
struct MyString(String);

impl fmt::Display for MyString {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "*{}", self.0);
        Ok(())
    }
}

fn main() {
    let a = MyString(String::from("jack"));
    println!("a = {}", a);
}
```

这里更多是一个编程技巧。



## 高级类型

### 用 type 起个短名

前面说到了使用 type 可以在 trait 中定义一个类型，留待实现者填上具体类型，和泛型不同，trait 里面的 type 只能实现一个。但这里讲的是 trait 以外， type 的其它用法。

最经典的莫过于可以给复杂的名字起个简短的名字了；

```rust
use std::sync::Arc;
use std::sync::Mutex;

struct Person {
    name: String,
    age: u32,
}

type ArcMutex<T> = Arc<Mutex<T>>;

fn new_arcmutex<T>(t: T) -> ArcMutex<T> {
    Arc::new(Mutex::new(t))    
}

fn main() {
    let a = new_arcmutex(Person {
        name: "jack".to_string(),
        age: 30,
    });
}
```

我们在多线程中经常用到多个线程共同所有切需要修改某个对象的场景，当然就需要使用 Arc\<Mutex\<T\>\> 这样的 类型，因此为了少打尖括号，使用 type 起个好打出来的名字也挺好，并且封装了 new 函数，省去每次连续打 new 的麻烦。

在很多标准类库中，也经常看到类似这样的定义：

```rust
pub type Result<T> = result::Result<T, Error>
```

这就是说，错误都是 Error，但 Ok 的情况下，返回值和 T 相关，这样错误类型的代码就可以少打一些。



### ! 表示不返回值

不返回值表示对某种情况下，不返回任何东西。该如何理解呢？比如：

```rust
fn main() {
    let mut x = 0;
    loop {
        x += 1;
        let a = match x {
            5 => x,
            _ => continue
        };
        println!("a = {}", a); 
        break;
    }
}
```

上面的代码中，初始化下，x = 0，进过 x += 1，x 变为 1，match 后发现不等于 5，于是 continue，此时 match 即不返回任何值，而是回到 loop 开头，a 的值为定义。直到 x = 5，a 才赋值为5。

函数可以声明为没有返回值的类型：

```rust
fn print_something() -> ! {
    println!("in print something");
    panic!("will panic")
}
```

这里不能删去 panic!，除非用其它不会让 print_something 返回的函数或者宏代替（比如写一个死循环），否则，只要 print_something 返回，那么就不是 ! 返回值，比如删去 panic!  的话，返回值就是空tuple：() 。

unwrap 就是这样的函数：

```rust
impl<T> Option<T> {
pub fn unwrap(self) -> T { match self {
            Some(val) => val,
            None => panic!("called `Option::unwrap()` on a `None` value"),
        }
} }
```



### Sized trait

思考一下 &str 的 str，每次用到它的时候，总是加上 &，因为 Rust 在编译的时候无法知道 str 的具体大小，加上 & 后， Rust 只要知道 str 的地址和其长度就行了，地址和长度（usize）都是在编译时就知道其所需内存空间的，因此 &str 就可以编译。

所以，Box\<str\> 和 Rc\<str\>  就不需要加上 &，因为它们是用 unsafe 代码直接使用指针来实现的，指针大小当然也是编译的时候就知道了。

任何一个泛型，都默认是 Sized trait的，即编译的时候就知道其大小：

```rust
fn print_something<T>(t: T) where T: Debug {
    println!("t = {:?}", t);
}

fn main() {
    print_something(Person {
        name: "jack".to_string(),
        age: 30,
    });
}
```

实际上，print_something 是：

```rust
fn print_something<T>(t: T) where T: Debug + Sized {
    println!("t = {:?}", t);
}
```

Sized 是默认的 trait，一般情况下都会被省略。那么，我们的确也可以指定一种实际上编译的时候还未能确定其大小的泛型：

```rust
fn print_something<T>(t: &T) where T: Debug + ?Sized {
    println!("t = {:?}", t);
}

fn main() {
    print_something(&Person {
        name: "jack".to_string(),
        age: 30,
    });
}
```

可以看到，我们把 Sized 改成 ?Sized 后，T 就表示一个编译时不能确定大小的泛型了，那么，此时就只能修改成引用类型了（不知道 T 的大小，但&T的大小是可以确定的，即地址是确定的）。



## 函数指针和闭包

### 函数指针

之前使用的都是闭包，但实际 Rust 是可以用函数指针的：

```rust
fn multiple2_if_odd(i: &i32) -> i32 {
    if i % 2 != 0 {
        return *i * 2;
    }
    *i
}

fn main() {
    let f = multiple2_if_odd;

    let a = vec![1, 2, 3, 4, 5, 6];
    let new_a: Vec<i32> = a.iter().map(f).collect();

    println!("a = {:?}", a);
    println!("new_a = {:?}", new_a);
}
```

当然，用闭包似乎更精炼一点。



### 闭包也是一个 ?Sized

和 str 一样，闭包也是一个编译的时候并不知道其大小的类型：

```rust
fn make_closure()  -> dyn Fn(i32) -> i32 {
    |x| x + 1
}

fn main() {
    let a = make_closure();
    println!("3 + 1 = {}", a(3));
}
```

Fn（类似的还有FnMut和FnOnce）在编译期间也是不知道其大小的，对于编译器不知道的类型，要么用引用要么用box（或类似容器）来存放即可，这里只能用box（或类似容器，如Rc），因为需要把闭包的生命周期从函数中延长至外部：

```rust
fn make_closure()  -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}

fn main() {
    let a = make_closure();
    println!("3 + 1 = {}", a(3));
}
```



## 宏

### 写一个自己的函数宏（也叫元编程）

函数宏是 Rust 最复杂的一个概念，即是官方教程，也没有给出很详细的例子，直接说大部分 Rust 程序员都只是使用函数宏而不是创建函数宏，因此没有详细讨论，当然如果真的很想学习 Rust 的函数宏，官方也给出了很详细的教程：

https://doc.rust-lang.org/rust-by-example/macros.html

这里只给一个非常简单的函数宏例子：

```rust
macro_rules! my_macro {
    ($a:expr) => {
        $a + 1;
    };
}

fn main() {
    println!("9 + 1 = {}", my_macro!(9));
}
```

上面的代码，macro_rules! 即将定义一个函数宏，函数宏名叫 my_macro，后面的大括号就是函数宏要干的内容。函数宏，其实就是匹配参数，匹配中的，就做对应的事情，很像 match，比如上面的代码中，匹配模式即为($a:expr)，expr表示第一个参数表达式，$a 为这个表达式的值，匹配中了，那么就做 => { ... } 里面的事情，做什么的呢， $a + 1，即 expr 表达式的值 + 1。

所以，my_macro!(9) 表达式 expr 为 9，即 $a 为9，也即 9 + 1为10，即 my_macro 函数宏返回 10。

实现任意参数输入，即反复匹配：

```rust
#[macro_export]
macro_rules! my_macro {
    ($a:expr) => {
        $a
    };
    ($a:expr, $($b:expr), *) => {
        my_macro!($a) + my_macro!($($b), *)
    };
}

fn main() {
    println!("{}", my_macro!(1, 2, 4, 5, 6));
}
```

以上即实现任意参数相加，其中 $($b:expr), * 即反复匹配，在第 6 行这个匹配模式中，先拿出第一个匹配出来，剩余的递归匹配，直到为没有参数位置，展开后即各个数相加。

其输出为：

```rust
18
42
```

符合预期。

注意最开头的 \#[macro_export] 表示这个宏可以导出使用。



### 过程宏

还记得我们给自己的 struct 加上 #[derive(Debug)] 的宏吗？这个就是过程宏，它帮我们不用写什么代码，直接就实现了 Debug trait，从而可以 println 出来，这就是过程宏。现在我们就写一个自己的过程宏。

首先新建一个 library，不妨就叫 mymacro 里面简单的定义一个 trait：

```rust
pub trait PrintSelf {
    fn print_self(&self);
}
```

我们最终要的效果即：

```rust
#[derive(MyMacro)]
struct Person {
    age: i32, 
    name: String,
}

fn main() {
    let a = Person {
        age: 99,
        name: "jack".to_string(),
    };

    a.print_self();
}
```

也就是说，只要在我们的 stuct 上加上 #[derive(MyMacro)]，那么就可以实现 PrintSelf trait ，不需要自己写太多代码。

因此，我们新建一个 library，约定俗成叫 mymacro_drive，在里面，首选要在 Cargo.toml 文件中增加依赖和告诉cargo我们要定义过程宏了：

```rust
[lib]
proc-macro = true

[dependencies]
mymacro = {path = "../mymacro"}
syn = "*"
quote = "*"
```

其中的 syn 库可以解析 Rust 代码，而quote 库可以生成 Rust 代码，在mymacro_drive的 lib.rs 文件中增加以下代码：

```rust
extern crate proc_macro;
use proc_macro::TokenStream;
use quote::quote;
use syn;

#[proc_macro_derive(MyMacro)]
pub fn my_macro_derive(input: TokenStream) -> TokenStream {
    let ast = syn::parse(input).unwrap();
    impl_my_macro(&ast)
}

fn impl_my_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let gen = quote! {
        impl PrintSelf for #name {
            fn print_self(&self) {
                println!("hello, i am {}", stringify!(#name));
            }
        }
    };
    gen.into()
}
```

其中，#[proc_macro_derive(MyMacro)] 表示定义了一个过程宏 MyMacro，一旦编译器遇到 #[derive(MyMacro)] ，就会调用 my_macro_derive函数。而TokenStream则是编译器解析给我们的 struct 流，syn::parse 则可以读取解析出来内容，内容即为 strut 的属性，第 14 行的quote 宏可以生成 Rust 代码，里面的 15 - 19 行其事是给 quote 生成代码用的，#name 即替代 13行的宏，stringify!(#name) 即转为字符串方便打印。

重新把代码编程 TokenStream 输出给编译器，编译器即可把 quote 里面的宏内容在使用 #[derive(MyMacro)] 的地方展开，从而少写了 quote 宏里面的代码。这就是过程宏的展开过程。

类似这样的还有属性宏，即展开后 struct 获得一些字段，还有类函数宏，即自行解析宏里面的代码，比如：

sql!(select name from table_name where id = 3);

也都是机遇 syn 和 quote这两个库来实现的。





