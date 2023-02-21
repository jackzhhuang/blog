---
title: 区块链（三）：挖矿难度和网络部署
date: 2023-02-21 13:14:27
published: false
tags:
categories:
- Rust
---

上一节讲了区块链的协议，这一节主要讲区块链的挖矿难度和网络部署。

<!--more-->

## 挖矿难度

前面讲了，区块链的安全是依靠挖矿难度来保证的，即依赖工作量证明（proof of work）来获取到写入区块链资格，只有算力很强的节点才能争取到写入区块链的权利。这里进一步细讲挖矿的难度。首先，为什么挖矿难度可以增加比特币的安全性呢？

假设如果区块链的挖矿难度很低，这样导致的结果就是很多人都可以在短时间内争取到写入区块链的权利，于是区块链就会很容易出现分叉：

![区块链分叉](https://www.jackhuang.cc/svg/blokchain-mang-forks.svg)

图中，大量红色区块追加在黄色区块中，这种分叉不利于区块链的数据一致性保证，这样就很难保证到底哪一个新节点能成为最长的区块链，很难达到最终一致的结果。而且也很容易造成 doubel spending 攻击。



### double  spending 攻击

所谓 double spending 攻击，就是恶意节点在完成一笔交易后，强行在交易节点前的同一个节点强行写入一个新节点，造成分叉，只要新节点最后变成最长链，根据前面讲的知识，区块链系统会接受最长链为正式节点，这样之前的交易就会被回滚，从而导致卖家损失，例如下图所示：

![double spending 攻击](https://www.jackhuang.cc/svg/double-spending-attack.svg)

假设 A 是一个恶意用户，第一次交易，A 给 B 转账，但是 A 作为一个恶意节点，它新创建一个自己的另一个账户 A‘，发起第二次交易，A 给自己的 A‘ 转账，这回交易的输入还是老区块（图中的黄色区块）的输出，那么，图中的蓝色区块交易就有可能会被回滚，特别是如果挖矿难度值低的时候，蓝色的交易区块被回滚的可能性就非常之大，因为恶意节点可以短时间内继续写入更多的区块在红色区块之后，即图中后面的灰色区块，最终使得蓝色区块因为不是最长区块而被回滚。若此时 B 给 A 发货了，那么 B 将造成损失。

提升难度值可以让出现这种攻击的概率变低。而且一般交易平台，也会等待自己的交易区块之后有 6 个区块写入后才确认交易，进一步降低了区块被回滚的概率。这个概率，根据数学家的计算，是非常接近 0 的，可以信任。（这个世界上没有 100% 分布式数据一致性算法，均是极大接近于 0 而已）



### 伯努利过程

整个区块链的挖矿其实是一个伯努利过程，即类似抛硬币，每一次抛到正面或者反面的概率都是 50%，不会因为抛了 9 次都是正面，那么第 10 次是反面的概率而变得更大。通用点说，伯努利过程是每一次随机试验都是独立的概率，而不会因为前面的结果影响下一次的概率变化。

因此，挖矿虽然比拼算力，但若算力相当的情况下，每一个矿工节点获得写入区块链权利概率都是一样的。因此，只要大部分节点都是善意节点，那么恶意节点想做恶的可能性就很低，而且，正如上一节所说，有超强算力的节点都是希望系统稳定的，不然，一个不值得信任的区块链是没有价值的。



### 挖矿难度的控制

挖矿难度是区块链安全的基石，那么区块链难度是怎么控制的呢？前面有讲，主要是靠区块头的两个字段：target 和 nonce，这里注意的是，nonce 字段只有 4 个字节，相较于哈希结果 sha256 的空间（2<sup>256</sup>）是很小的，因此，还需要区块的中的交易字段来增大哈希参数的输入空间。

还有要注意的是，难度值 difficulty 和 target 是反比关系，即 target 越大，难度越低，反之亦然。可见 target 前面的0越多，也就越难。

那么 target 是怎么控制的呢？中本聪首先他规定了，每隔十分钟要产生一个区块，每 2016 个区块就要产生一次新的 target（即每 14 天调整一次 target），新的 target 和老的 target 值关系如下：
$$
新 target = 老 target * \frac{实际时间}{期望时间}
$$
上面的公示很容易理解，当实际时间和期望时间相等的时候，那么说明难度值可以不需要改变，因为我们的期望值就是每 14 天调整一次，如果实际时间不足 14 天，那么<font size=5> $\frac{实际时间}{期望时间}$ </font>会小于 1，此时新的 target 就会变小，导致难度值变大。相反如果实际时间比期望值长，说明难度变高了，此时 <font size=5>$\frac{实际时间}{期望时间}$ </font>会大于 1，那么新的 target 变大，难度值就变小了。

调整难度值的意义在于我们的算力一直都在增强，如果保持一成不变，10 分钟出一次新区块的频率就难以保证。



## 区块链网络部署

区块链的网络部署是基于 p2p 网络的，其网络特点如下：

1、任何节点都是随意加入和离开区块链 p2p 网络；

2、节点的传输不依赖实际物理拓扑规则，有可能两个较远的节点也可以成为邻居节点；

3、区块链的块大小一般为 1 MB；因为区块链这种数据量大的数据库，如果每一块都很大，会非常的耗网络带宽，一些轻节点实际只是关心某一些交易，因此大块的区块是不利于在 p2p 网络传输的，小块传输可以增强传输的可用性；

### 区块链的节点

区块链的网络节点部署大致可以分成四类：

1、全节点；