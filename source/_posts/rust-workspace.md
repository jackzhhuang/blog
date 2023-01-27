---
title: Rust的工作空间（workspace）
date: 2023-01-26 17:27:20
tags:
- Rust
- workspace
categories:
- Rust
---

前面的例子，都是通过cargo new命令来创建的单一工程，但实际开发中，每一个模块都是由多个单独的领域组合起来构建的，因此这里引入workspace概念。有了workspace概念，一个项目才可以方便的划分成不同的领域，每一个领域能做到单元测试，单独构建，而整个workspace又能做集成测试，集成构建。这样，单人工作和团队协作才能有机结合起来。

<!--more-->

## 为什么需要workspace

正如前面所说，项目中，个人既要能做到对自己的代码进行单元测试，也要能和别人的代码进行集成测试，因此，项目必须能同时满足独立构建和协同构建。Rust 的workspace正是这个概念。它可以创建独立的 library 工程，也可以同时对几个 library 工程进行依赖构建，也即独立工作时可以非常独立，需要协作时又能迅速集成，期间不需要太多复杂的操作，仅仅只是从工程子目录切换到工程workspace目录，简单而有效。



## 如何构建workspace

我们以 https://github.com/jackzhhuang/rscount 为例，说说workspace的结构是怎么样子的。

首先，切换到 workspace_example 分支，因为master分支还在开发中，可能目录结构等等都会变化，为了说明 workspace 结构，我新拉了一个示例分支用来讲 workspace 的结构。

拉取分支后，可以看到，其目录结构如下：

```rust
rscount
├── Cargo.lock
├── Cargo.toml
├── LICENSE
├── README.md
├── rsconfig
│   ├── Cargo.toml
│   └── src
│       ├── config.rs
│       └── lib.rs
├── rscount_main
│   ├── Cargo.toml
│   └── src
│       └── main.rs
└── rsfile
    ├── Cargo.toml
    └── src
        ├── lib.rs
        ├── rs_code_dir.rs
        └── rs_code_file.rs
```

首先 rscount 就是我们的workspace根目录，这个目录是用mkdir命令创建的，cd进入rscount就可以开始workspace的创建了。

需要说的是，rscount/Cargo.toml 是 workspace 的结构说明，里面主要展示 workspace 都有哪些子工程，该文件内容是手动写入的，每创建一个子工程就需要手动写入一次工程名：

```rust
[workspace]
members = [
    "rscount_main",
    "rsfile",
    "rsconfig",
]
```

可以看到，一共有三个子工程，名字分别是：rscount_main，rsfile和rsconfig。这些子工程都是用 cargo new 命令来创建的。对于 rscount_main 这种执行文件工程，直接在rscount目录下执行：

```rust
cargo new rscount_main
```

即可。

而对于rsfile和rsconfig，因为是 library 工程，需要加上 --lib参数：

```rust
cargo new rsfile --lib
cargo new rsconfig --lib
```

创建好这些工程后，就可以写代码编译了，当然可以一起编译，即在rscount下cargo build，也可以分别在各个子工程下分别编译。这里还需要注意的是，rscount_main依赖了rsfile和rsconfig两个工程，因此需要在其rscount_main/Cargo.toml中告诉编译器这个依赖关系：

```rust
[dependencies]
rsfile = { path = "../rsfile"}
rsconfig = { path = "../rsconfig"}
```



## 设置编译选项-编译release版本

编译的时候，cargo build加上 --release参数即可编译release版本，我们还可以指定release版本的参数，比如最常见的优化级别，debug版本优化级别我们设置为0，release版本优化级别设置为3:

```rust
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

还有很多编译选项可以甚至，具体可以参考官方文档。



## 使用crates.io

除了自己写 library，还可以求助crates.io，上面有很多其他人编写的 libray库，例如我们需要mysql客户端库，但我们不想自己写一个，肯定有人已经写好了，于是我们上crates.io搜索mysql： ![使用create.io搜索第三方库](https://www.jackhuang.cc/images/WX20230127-101853@2x.png)

​	将mysql这个库写入dependencies中：

```rust
[dependencies]
rsfile = { path = "../rsfile"}
rsconfig = { path = "../rsconfig"}
mysql = "*"
```

然后编译，可以看到cargo可以自动下载库并建立依赖，直接使用了。

