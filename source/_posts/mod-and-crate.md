---
title: Rust的模块化管理
date: 2023-01-10 23:18:55
toc: true
tags:
- Rust
categories:
- Rust
---

任何语言都离不开一个基本概念——包，Rust当然也不例外。按照官方文档教程说法，Rust最基本的包叫crate，它包含了一个或若干个library文件（库文件）或者binary文件（执行文件）。

<!--more-->

## 1. 从最简单的包演化说起

任何项目都不可能一个main文件写到底，我们总是要分成几个文件，甚至还要给文件放进不同目录，甚至会事先编译一些文件来管理项目文件。为了学习Rust的相关概念，我们从最简单的一个文件说起，慢慢把它演化成我们常见的项目管理方式，首先来个最简单的main.rs：

```rust
#[derive(Debug)]
struct Person {
    name: String,
    age: u32,
}

fn main() {
    let p = Person {
        name: "jack".to_string(),
        age: 100
    };

    println!("p = {:?}", p);
}
```

这个main.rs文件没有任何模块化管理的概念，Person结构体直接就能用。

### 1.1 第一个mod

现在我们把它放到一个module里面吧：

```rust
mod PersonData {
    #[derive(Debug)]
    struct Person {
        name: String,
        age: u32,
    }
}
```

这个就是我们第一个module了，这里的module很像C++的名字空间（但也不完全是），因此，想使用Person就必须加上PersonData：

```rust
fn main() {
    let p = PersonData::Person {
        name: "jack".to_string(),
        age: 100
    };

    println!("p = {:?}", p);
}
```

可惜这样是编译不通过的，因为任何一个mod里面的东西（也就是module），包括struct，字段，方法，函数甚至子mod都统统是私有的，想对mod外面暴露，就必须定义为pub，因此，我们给Person等名前都加上pub，意味共有的，可以个mod的外部使用：

```rust
mod PersonData {
    #[derive(Debug)]
    pub struct Person {
        pub name: String,
        pub age: u32,
    }
}
```

 这个时候Person以及他的字段都变成pub了，外部可以使用了。代码可以通过编译。

可见，一个mod关键字，可以声明一个module。



### 1.2 把mod放到别的文件

以上代码我们都是在main.rs文件里面写的，显然我们应该独立出来，假设独立出来的文件名就叫PersonData.rs，那么我们不需要在文件中声明mod PersonData了（如果声明了mod PersonData，那么就是PersonData的子包也叫PersonData）：

```rust
#[derive(Debug)]
pub struct Person {
    pub name: String,
    pub age: u32,
}
```

此时，main.rs的改动就是mod一下这个文件名，从而引入Person结构：

```rust
mod PersonData;

fn main() {
    let p = PersonData::Person {
        name: "jack".to_string(),
        age: 100,
    };

    println!("p = {:?}", p);
}
```

其它地方并没有什么改变。

可见，一个rs文件，可以被当作一个module。



### 1.3 把mod放到别的目录

如果文件PersonData.rs放到别的文件夹呢？假设我们把PersonData.rs移动到DataSet目录里面，那么，此时目录结构如下：

```rust
├── DataSet
│   └── PersonData.rs
└── main.rs
```

此时，main.rs怎么找到PersonData这个模块呢？

有两种方法：

1、和main.rs同目录建一个DataSet同名的rs文件，在里面pub mod PersonData，pub表示这个DataSet包要公开PersonData这个module了。

DataSet.rs内容如下：

```rust
pub mod PersonData;
```

此时，目录结构如下：

```rust
.
├── DataSet
│   └── PersonData.rs
├── DataSet.rs
└── main.rs
```

而main.rs中，mod DataSet（此时不需要pub mod，因为和main.rs是同一个文件，表示引入DataSet），且由于Person被放到了DataSet目录的PersonDate.rs文件中，因此需要修改一下访问它的路径了：

```rust
mod DataSet;

fn main() {
    let p = DataSet::PersonData::Person {
        name: "jack".to_string(),
        age: 100,
    };

    println!("p = {:?}", p);
}
```

可见，如果一个目录想变成module，就必须在目录同一级目录中，创建一个相同名字的rs文件，并通过它来里面的pub语句来决定公开目录中哪个module。

第二种方法，我个人认为更简便，即在目录中创建mod.rs，里面干和上面DataSet.rs同样的事情，即决定对外公开目录中哪个module。

此时目录结构如下：

```rust
.
├── DataSet
│   ├── PersonData.rs
│   └── mod.rs
└── main.rs
```

其中，mod.rs的内容和之前第一种方法的DataSet.rs内容一样。这两个方法都是等效的。

可见，目录想变成module，还可以在目录中创建mod.rs文件来达到这个目的。



## 2.  模块的路径

### 2.1 根目录、绝对路径和相对路径

前面我们演化并展示了，一个struct类型如何变成不同位置的mod，以及我们的main.rs如何引用它。那么，Rust这些模块路径是通过什么原则去寻找的呢？

回顾一下，我们新建一个Rust工程的时候使用的是cargo new命令，例如，cargo new greeting，即创建了名为greeting的工程，进入greeting目录，tree命令可以看到创建的工程结构：

```rust
.
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── DataSet
│   │   ├── PersonData.rs
│   │   └── mod.rs
│   └── main.rs
```

其中src是自动生成的，main.rs是自动生成的，而 DateSet目录是后来添加的。

Cargo.toml则是工程描述文件，包含版本，工程名已经工程的外部依赖项。

需要编译编译的时候，在greeting目录中，执行cargo build就可以生成二进制文件。如果想生成库文件，则需要加上--lib命令选项，且不需要写main函数。

这里主要讲的是，在用cargo new生成工程的时候，src目录即为改工程的根目录。在使用mod关键字去引入工程时，会以src为起点作为根目录，去寻找引用的模块。根目录其实可以用crate来代替，例如上面对Person的访问可以写成：

```rust
    let h = crate::DataSet::House::HouseData::HouseInfo {
        address: String::from("shenzhen"),
        no: 8000,
    };
```

除了从根目录找，当然还会找同文件的mod模块，即本文一开始写的第一个mod，它写在main.rs文件中。

还有一个寻找路径就是相对路径，即如果在子目录，那么，也会从子目录开始寻找引用的模块。例如我们新增一个House模块在DataSet目录下，并在Hous模块中添加HoseData.rs文件，文件内容如下：

```rust
#[derive(Debug)]
pub struct HouseInfo {
    pub address: String,
    pub no: u32, 
}
```

此时，我们的根目录开始的tree命令输出是这样的：

```rust
.
├── DataSet
│   ├── House
│   │   ├── HouseData.rs
│   │   └── mod.rs
│   ├── PersonData.rs
│   └── mod.rs
└── main.rs
```

那么如果我们在DataSet模块输出House模块路径是怎么样的呢？直接以DataSet目录为起点（即相对路径）输入出House模块即可：

```rust
pub mod PersonData;
pub mod House;
```

 这样DataeSet模块就可以对外输出House模块了。当然House模块也需要对外输出它的HouseData模块，即House目录下的mod.rs文件需要：

```rust
pub mod HouseData;
```

因为任何模块里面的东西都是私有的，需要显示的说它是pub的才行。

此时我们在main.rs中就能直接访问House模块了：

```rust
mod DataSet;

fn main() {
    let p = crate::DataSet::PersonData::Person {
        name: "jack".to_string(),
        age: 100,
    };

    let h = crate::DataSet::House::HouseData::HouseInfo {
        address: String::from("shenzhen"),
        no: 8000,
    };

    println!("p = {:?}", p);
    println!("h = {:?}", h);
}
```

总结，Rust的寻找路径可以这样：

1、从根目录开始寻找；

2、从相对路径开始寻找

3、从当前兄弟模块开始寻找。

根目录即src目录，在代码中可以省略或者以crate开头。

### 2.2 使用use 

但如此长的名字确实不好打，Rust给我们提供了一个关键字use，我们只写一次就可以：

```rust
use DataSet::House::HouseData::HouseInfo;
```

这样，我们的h指针在初始化的时候就不需要写这么多层级了：

```rust
    let h = HouseInfo {
        address: String::from("shenzhen"),
        no: 8000,
    };
```

### 2.3 使用pub use和as

设想如果DataSet有更多更深的模块的话，我们的每次使用这些又深名字又长的模块都要写一次use，实在太麻烦，此时我们可以在DataSet里面使用pub use来直接吧HouseInfo导出去：

```rust
pub mod PersonData;
pub mod House;
pub use House::HouseData::HouseInfo;
```

用了上面第三行的pub use，HouseInfo就如同DataSet的一个struct一样被导到外面去了：

```rust
mod DataSet;

use DataSet::HouseInfo;

fn main() {
    let p = crate::DataSet::PersonData::Person {
        name: "jack".to_string(),
        age: 100,
    };

    let h = HouseInfo {
        address: String::from("shenzhen"),
        no: 8000,
    };

    println!("p = {:?}", p);
    println!("h = {:?}", h);
}
```

在用，不管子模块有多深，只要使用一次pub use，后续都不需要重新use一遍长路径。

在使用use的时候还可以给导出的struct或者模块起个新名字：

```rust
use DataSet::HouseInfo as DHI;
```

  此时，使用HouseInfo的时候我们可以用DHI代替了。

### 2.4 使用super

和crate不同，crate是表示根目录，而super则表示当前目录的上一级目录。

例如我们在House中想使用PersonData：

```rust
use super::super::PersonData::Person; 

#[derive(Debug)]
pub struct HouseInfo {
    pub address: String,
    pub no: u32, 
    pub someone: Person,
}
```

  这里用了两个super，从之前的tree看到，第一个super是PersonData.rs的父目录House，第二个super就是House的父目录DataSet，而PersonData.rs就在DataSet下。我们当然也可以用crate目录（也即根节点去引用PersonData）：

```rust
use crate::DataSet::PersonData::Person; 

#[derive(Debug)]
pub struct HouseInfo {
    pub address: String,
    pub no: u32, 
    pub someone: Person,
}
```



## 2.5 pub struct和pub enum

最后要提的是，pub struct和pub enum不同点：

pub struct只是把struct导出，它的字段，方法还需要再pub来确定是否导出。而enum一旦pub了，它的字段都会变成共有的，都会被导出。



## 3. 同时引入多个模块

如果需要引入一个模块中多个struct或者其它什么的，可以用大括号来在一行中表示，例如，我们既需要Person也需要HouseInfo，除了写两行use外，还可以这样：

```rust
use DataSet::House::HouseData::HouseInfo;
use DataSet::PersonData::Person;
```

改写成：

```rust
use DataSet::{House::HouseData::HouseInfo, PersonData::Person};
```

 甚至一个*号来表示所有的可以导出的DataSet的东西：

```rust
use DataSet::*;
```

 但星号还是谨慎使用，因为比较随意，可能不经意引起名字冲突。



## 4. 总结

这一节从main.rs分裂出一个子模块，并把它放到一个目录中，再新建一个子模块到子子目录中，并通过使用mod（引入一个模块），pub（声明struct，方法，字段，enum为公有，或者用来导出struct，函数或者enum）来控制访问模块。

使用use来引入复杂的层级，使用pub use来导出一个复杂的层级，还可以使用as来给导出或者引入的模块或者类型定义一个新名字。

再简化总结一下：

一个crate可以有多个模块，crate的根在src目录。

使用方：用mod+use来引入模块，mod引入顶层模块，use引入模块中更复杂的层级结构。如果是引入父级目录，则不需要mod，直接use即可。

导出方：用pub导出字段，struct，方法函数等，用pub use导出层级深的目录结构。

as可以提供改名的便利。

