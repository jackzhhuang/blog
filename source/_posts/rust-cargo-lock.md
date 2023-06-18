---
title: 记一次踩 Cargo.lock 的坑
date: 2023-06-17 22:43:02
tags:
- Rust
---

“Cargo.lock 是自动生成的，可以删掉重新编译一次就好了。”

“我们无需关心 Cargo.lock，我们控制好 Cargo.toml 的内容就好了。”

<!--more-->

最近编译 Rust 工程，突然报错，具体表现是 serde 解析 alias 和 rename 的时候其中一个会返回 None。诡异的是，原来的版本并没有问题，对比 Cargo.toml 文件引用 serde 的版本也都是一样的。简单说来，即：

一样的 toml 文件，一样的代码，一样的机器，一样的环境，但编译运行出来的代码是不一样的。

最后发现定位居然是没问题的工程引用的 serde 是 1.0.152 版本的，而有问题的工程引用的 serde 是 1.0.164。那么为什么后者会引用更新的版本呢？

于是尝试删掉本地 serde 库，让 Cargo 重新拉取 serde。结果：

原来引用 1.0.152 版本的工程还是引用 1.0.152，而原来引用 1.0.164 版本的工程还是会引用 1.0.164。太诡异了。到底是谁在控制补丁版本号呢？

于是尝试询问 chatGPT，得到以下答案：

> 问：这个选择是rust自己控制的吗？如果我们不指定补丁版本号的话，所以才会出现之前说的那种情况，虽然我指定都是1.0版本，工程也一样，但就是有一个选择了1.0.152，另一个选择了1.0.164，到底什么会影响这个选择？
>
> 答：对，这个选择是由 Rust 的包管理器 Cargo 自己控制的。Cargo 选择特定版本的原则是尽可能选择最新的版本，但同时满足所有依赖关系的约束。
>
> 如果你在 `Cargo.toml` 中指定 `serde_derive = "1.0"`，那么 Cargo 可以选择任何主版本号为 1 的版本，无论次版本号或补丁版本号是多少。所以，如果在你获取或更新依赖的时候，serde_derive 的最新版本是 1.0.152，那么 Cargo 就会选择这个版本。如果后来 serde_derive 发布了新的版本，比如 1.0.164，那么新的工程在获取依赖的时候，Cargo 就会选择这个新版本。
>
> 影响 Cargo 版本选择的因素包括：
>
> 1. 在 `Cargo.toml` 中明确指定的版本。比如 `serde_derive = "1.0.152"` 就只会使用版本 1.0.152。
> 2. 其他依赖库的版本需求。如果你的项目还依赖其他库，而这些库又依赖 serde_derive，并且指定了特定版本，那么 Cargo 将在满足所有库的版本需求的同时选择一个版本。
> 3. 当你获取或更新依赖的时间。如上所述，如果 serde_derive 在你获取依赖的时候发布了新版本，那么 Cargo 就会选择新版本。
> 4. 在 `Cargo.lock` 文件中锁定的版本。`Cargo.lock` 文件记录了上一次构建时所有依赖的具体版本。如果 `Cargo.lock` 文件存在，并且你没有改变 `Cargo.toml` 或运行 `cargo update`，那么 Cargo 就会使用 `Cargo.lock` 中记录的版本，而不会获取新版本。
>
> 如果你希望所有工程都使用同一版本的 serde_derive，你可以在 `Cargo.toml` 中指定具体的版本，比如 `serde_derive = "1.0.152"`。或者，你可以定期运行 `cargo update` 来确保所有依赖都是最新的版本。

关键就是上面的第三和第四点，即：若Cargo.toml 文件没有指定更细的版本号，那么Cargo.lock 在生成的时候会记录当前最新的版本作为编译版本，在未来，即使库有所更新，但都会以 Cargo.lock 为准。这么说的话，Cargo.lock 就像是一个编译版本库的快照文件。

如果我们删除 Cargo.lock，相当于给 cargo 一次重新选择版本号的机会，大多数情况下，这没什么问题，因为我们基本上都会在 Cargo.toml 文件中指定大版本号，而补丁版本往往能做到向前兼容，因此即使我们删掉 Cargo.lock 重新生成，补丁版本号更新了，我们也不会有所察觉。但如果这个兼容没做到位，就会出现这次的情况，即 Cargo.toml 完全一样但却不是引用同一个版本的问题。

解决方案当然是修改 Cargo.lock 文件，使之指定到具体的补丁版本，当然也可以修改 Cargo.toml 文件直接指定到更细的补丁版本。

不管怎么做，都提醒一点：尽管 Cargo.lock 文件是自动生成文件，但我们不能忽视它的内容，作为编译的快照文件，它非常重要，很多细节隐藏在里面，若仅仅盯着 Cargo.toml 文件，其实际情况未必就是我们想当然的那样。有时编译失败，不妨考虑考虑 Cargo.lock 的内容是否就是自己想的那样呢？