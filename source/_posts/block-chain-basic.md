---
title: 区块链（一）：基本结构
date: 2023-02-12 20:47:38
tags:
categories:
- Blockchain

---

今天我们就开始总结一下区块链方面的知识吧，从最基本的区块链结构讲起。

<!--more-->

## 区块链基本信息

顾名思义，区块链，即一个个块连接起来的链表，但的确加了不少附加信息在上面，我们先看三个最基本的信息：

1、创世区块；

2、区块的高度；

3、区块链难以修改的特性。

我们先直观的看一下区块链的样子吧：

![](https://www.jackhuang.cc/svg/blockchain-first-view.svg)

我们最先看到的是这是一个链表，但要注意，这个链表的增长方式是从创世区块开始的，也就是区块链和普通链表不一样，是反向增长的，即新增的区块指向最后一个区块，第一个区块是图中的创世区块。

创世区块是一个很特殊的区块，它一般硬编码在代码中，万事总是要有一个起头，所以创世区块作为区块链的头，要特殊编码，再往后，每一个区块的加入都必须遵循区块链的规则了。

区块中会分为区块头和与之相关的区块交易体，我们先关注一下区块头。可以看到每一个区块都有一个区块头，即上图红色框框部分，其中一个字段是前一个区块头的sha256结果，也即，每一个区块的标识符都用区块头的sha256来表示，这个结果会写在下一个区块的区块头的字段中，这样，这些区块就组成了一个有序且难以更改的区块链了。

有序，是因为每一个区块的头部字段都记录前一个区块的头部sha256，因此区块顺序不能变。

难以修改，是因为区块头一旦被修改，那么其sha256也被跟着改，也即区块的标识符被修改，这会影响下一个区块的区块头内容，也就导致下一个区块的标识也要修改，于是如此串联下去，一处改则处处改。随着区块的增多，修改难度会越来越大，这也就是难以修改的原因。也可以看出，越是古老的区块，修改的难度越大。

尾部的区块它的头部sha256值，也即其标识符可以保存起来，这样，我们就可以验证整条区块链是否有被修改。例如轻客户端（比如手机）很难去保存所有的区块，但它又想验证最近几个区块是否被修改，这样只需要从全区块链那里获取最近几个区块的头部，然后验证区块链标识是否和所持有的区块链标识是否相等，即可知道这些区块是否被修改，是否信得过了。

最后讲一下区块链的高度，区块链的高度即区块的多少，当然要注意，创世区块的高度是 0，也就是说，如果有 n 个区块，那么其高度是 n - 1。我们引用一个区块时，除了可以用它的标识符以外，还可以用它的所处高度来表达。例如高度为 0 的区块即创世区块。



## 区块头部

![区块头部](https://www.jackhuang.cc/svg/blockchain-head.svg)

上面的头部是被我们简化的，区块链的头部不仅仅有前块标识字段，还有其它字段，我们放大看看：

首先是版本，似乎没什么可说的。

接着就是前面提到的前块区块的头部 sha256，即其区块标识。

默克尔（merkle）值，是用于检验交易数据的，我们马上讲到。由于有了默克尔值，后面的交易数我们就可以校验，这样我们标识一个区块的时候只需要头部sha256即可，并不需要整个区块 sha256。

随机难度值和随机数是用于挖矿的，我们将来讲到。



## 默克尔树

### 结构

当我们有很多交易数据时，我们怎么校验它们是否被修改信得过呢？一种直观的方法当然是把它们放在一起然后算一次 sha256，如果有 n 个交易，那么算法时间和空间复杂度为 O(n)。

默克尔树提供了一个更快更省空间的方案，即将每一笔交易数据做一次 sha256（比特币中是连续计算两次 sha256，以下简称算 sha256 不特地强调），然后这些结果两两做一组再算一次 sha256，再两两做一组算 sha256，如此循环，最后只剩下一个 sha256 值，这个值就是默克尔树根值，有了这个值，我们就可以很快的校验交易是否被修改。其计算过程和结构如下：

![默克尔（merkle）树](https://www.jackhuang.cc/svg/merkle-tree.svg)

从上面看出，默克尔树树叶节点即我们的交易数据，往上每一层都是下一层数据两两结合计算 sha256 的结果，直到我们算出默克尔树根为止。

### 校验交易

我们需要校验某个交易是否被修改时，尤其是手机这种轻节点，只需要从全节点处拿到从下往上算的路径上的 sha256 节点即可去验证某个交易是否在这个区块中或者是否被修改过。

例如我们需要计算下图红色节点是否在区块交易数据中，轻节点只需要请求全节点下发黄色 sha256数据给它，它依次从下往上计算出蓝色节点的 sha256 值，直至计算出默克尔树树根值，对比区块头的默克尔树根值是否一致就可以知道交易是否在此区块中（区块头的sha256 我们是由区块标识符保证不被串改的，前面已经讨论）。

![默克尔树校验交易过程](https://www.jackhuang.cc/svg/merkle-tree-verify.svg)

由于不需要全节点所有的交易数据，此时算法复杂度和空间复杂度只有 O(log<sub>2</sub>n) 了。
