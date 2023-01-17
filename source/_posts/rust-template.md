---
title: Rust的模版编程
date: 2023-01-17 13:13:33
published: false
tags:
categories:
- Rust
---

从今天开始，要开始学习Rust比较高级的内容了，为了更细化，每个特性都单独为一节。

今天要讲的是Rust的模板编程。模板编程实际上并不少见，不论C++还是Java，只要使用容器，基本都离不开模板。Rust的模板和C++很相似，都是编译时根据模板代码生成实际的（concrete）代码然后再编译成二进制。

那么，今天就好好看看Rust的模板编程吧。

<!--more-->

<!-- toc -->

