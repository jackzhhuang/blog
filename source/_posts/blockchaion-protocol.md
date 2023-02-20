---
title: 区块链（二）：区块链的交易协议
date: 2023-02-18 20:24:26
tags:
categories:
- Blockchain
---

本节主要讲区块链的交易协议细节，即一笔交易是如何写入区块链的交易数据中的。

<!--more-->

## 去中心化的交易需要解决的问题

我们先看看中心化的交易是怎么样的。中心化的交易，货币是靠央行发行的，发行货币和发行多少货币一切都是它说的算，交易的清结算也是以它的账簿为准，任何交易数据只要写进它的账簿那么就是生效的。相比下，去中心化的交易系统因此有两个问题需要解决：

1、如何发行货币；

2、交易怎么进行才能做到可信。

因为是去中心化的，谁都可以有机会去更新区块链的数据，那么，货币发行怎么控制呢？同样，因为谁都可以有机会去写入交易数据，那么谁写入的数据才算是真实有效的呢？这些问题就需要区块链的交易协议去解决。

为了讲清楚交易协议，我们会提出一个个问题，然后看看区块链的去中心化交易协议是如何解决的。



## 区块链中的交易过程

抛开货币的发行问题，先看看交易是如何做到可信的，即一笔交易数据，如何保证双方都是出于真实意图的交易呢？

区块链两大加密技术之一就是公私钥，公私钥对就代表了一个区块链账户。其中公钥用于个人 ID 的标识，私钥用于签名，保证某一笔交易行为是这个人的意愿。我们举个例子，假设 A 需要转账给 B 和 C，那么交易记录就需要至少记录：

1、使用 A 的私钥签名交易数据，保证 A 的转账行为是 A 真实发起的。这个签名大家都认可，因为 A 的公钥是公开的，可以验证 A 的私钥签名；

2、交易记录中还需要 B 和 C 的公钥，因为此时想表达 B 和 C 这两个身份，就必须用他们的公钥，公钥此时就相当于区块链的身份 ID；

3、交易货币数量。

似乎以上数据就够了，但试想这样一个场景：

一个恶意用户，自己生成了公私钥对，告诉 B 和 C 说他是 A，那么，按照上面的三条数据要求，这个恶意用户是可以完全自己伪造出 A 转钱给 B 和 C 用户的。因此，这里就需要一些额外的信息，保证 B 和 C 是知道对方是不是 A。自然想到的是验证 A 的公钥，但 B 和 C 如何验证恶意用户出示的 A 的公钥是否真的是 A 的公钥呢？

这就需要上一节讲的区块链性质了——区块链上的信息都难以修改，因此是可信的。B 和 C 看到恶意用户出示 A 的公钥后，会去查区块链对应的公钥信息，即对应的余额，看是否有足够的钱支付本次转账，如果足够，那么这次交易就是安全的，这样，哪怕恶意用户不是 A，但他出示的公钥在区块链上的确有足够的余额，那就没问题。

因此，区块链只认公私钥对，只要交易时有足够的余额，那么交易就可以促成，是不是 A 也就无所谓了。

上面的文字，如果画成图形，那么看起来就是这样：

![交易时，需要校验身份和资金来源](https://www.jackhuang.cc/svg/blockchain-deal-verify.svg)

简单来说，就是每次交易，都需要能完成以下重要步骤：

1、出示付款发公钥和收款方的公钥；

2、付款方的公钥在区块链中能被找到；

3、在区块链中对应的付款方公钥下的余额足够支付本次交易；

4、付款方的公钥能验证交易签名。

校验通过后，交易信息将被写到区块中，交易就正式生效。如果此后 B 要转账给 D 和 E，那么同样也会进行上面的操作，画成图形，看起来像这这样：

![再一次交易](https://www.jackhuang.cc/svg/blockchain-deal-verify-another.svg)

由此可见，每一次交易，都有输入和输出双方，每一个输入都是上某次的输出，每一个输出都是下某次的输入，这样就保证的“输入 = 输出” ，账目是平的。



## 记账权的争夺——共识机制

以上，当然会有不少问题，即，因为任何人都可以往区块链写入交易数据，那么，

1、到底谁去做这个写入操作呢？

2、如果大家都写，那么势必会造成数据不一致，如果数据不一致了，那么又怎么做到以谁写入的为准呢？

3、如果有恶意用户争夺到写入权但拒绝写入某个交易从而使某个人无法完成交易怎么办呢？

这里就需要获得写入权的争夺了。在上一节讲到，区块头部有两个字段，一个是“目标难度值”，即 target，一个是随机字符串，即  nonce，它们就是用于控制记账权争夺的。

其算法思想如下：为了获取记账权，必须找到一个 nonce值，使得头部的 sha256 值小于 target。由于找到这样一个 nonce 值计算出的  sha256 需要枚举 nonce，因此对争夺记账权的 就依赖于争夺者的算力，算力越高，获得记账权的机会就越大。

这种依赖算力而不依赖投票的算法，我们叫做比特币的共识机制，它有一个好处就是规避了恶意用户制造大量僵尸账户进行投票影响正常结果的攻击。当然有人会问，如果恶意用户有很强的算力怎么办，要知道，算力很高的用户意味着有很强大的计算机系统，绝非个人所能做到，而拥有强大计算机系统的用户都是希望系统稳定的，就好像一个有权势的国王，他肯定是希望他的国家泰平，不然他自己也就当不成国王了，所以，需要算力极强的计算机系统用户才能写入区块链，保证了区块链的稳定，单个恶意用户是无法撼动区块链的。

总之，只要找到了 nonce，满足 sha256(区块链头部) < target，那么这个用户就可以写入区块链，他就会把结果广播给其它全节点。其它节点很快就可以验证他的结果，从而接受他的写入结果。



## 铸币交易

前面提到，交易的时候，算力高的用户会写入区块链，一般都是算力高的用户，他们绝非等闲之辈，因为能有这么高算力的人，现实中肯定也是很有能力的（不管是金钱还是权利），他们希望区块链系统稳定，以获取更多的好处，这个好处之一，就是铸币权。

铸币权就是每次写入交易时，都可以收取交易的手续费和创造出额外的货币，这类似中心化的央行印钱。手续费不用说，额外的货币即每次都可以往区块链的池子里加更多的余额，这个余额不需要像转账那样通过上一次的输出获得，而是直接创造出来，只需要记录记账者本人的公钥和余额即可。像比特币的规则，初始阶段，每次写入交易都可以创造出 50 个比特币，而后每 21 万个区块后则减半，即只能创造出 25 个比特币了，然后如此循环递减，直至为 0。

这就是区块链货币创造货币的规则。可见，区块链中主要有两种交易，一个是前面提到的转账交易，一个就是这里说的铸币交易。



## 区块链的分叉

前面说了通过算力来解决争夺的两个问题（记账权提出的 1 和 3 问题），问题 2 如果有两个算力都很强大的用户都算出 nonce，都往区块链中写怎么办？

一种情况，是往中间写，例如下图中，红色节点也是一个找到对应 nonce 值的节点，也打算往区块链中写入，但它的前置节点是在区块链的中间而不是末尾，按照记账权的规则，它是有权写入的，但又很明显，如果允许红色节点写入，那么原本黄色节点后面的蓝色节点将会失效，这对于已经确认交易的用户来说是不可接受的。

![区块链的分叉](https://www.jackhuang.cc/svg/blockchain-fork.svg)

对于这种想在中间插入的动作，由于其比较短，因此会被认为是孤儿节点（orphan block），这样的孤儿节点是不会被接受的。采用最长的分叉区块链是防止恶意用户插入恶意节点的方法。那么等长的插入呢？即同时都想往区块链中的最后一个节点插入新的区块，比如下图两个红色节点都想往最后一个黄色插入：

![等长区块插入](https://www.jackhuang.cc/svg/blockchain-fork-end.svg)

此时就属于分布式事务中的未决事务，即只能同时存在两个交易数据，那么什么时候能解决这个分叉呢？一般认为，某一个分叉后有超过 6 个区块节点，则认为另一个分叉的区块无法再超越它，因此短的分叉将会被大家舍弃掉（这种结论，是基于概率学得出来的，实际上，世界上没有 100% 能保证事务最终一致的设计，或者说数学上无法证明某个算法设计能 100% 保证数据最终一致，但实际经验告诉我们反超概率趋近于 0 的设计是可靠的），因此，区块链交易时，收入方会等待 T + 1日，保证交易已经写入区块链中且有了足够长的深度才会放心发货给出款方，这种在交易数额很大的时候会比较常见。