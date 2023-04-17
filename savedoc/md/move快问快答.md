# 账号地址的关系

move 中账号是被系统（move 或者 底层区块链，如 starcoin）verify 过的实体，代码以 account: signer 出现，sender 只能访问自己的账号。

地址包含在账号下，任何资源都是以其定义的账号地址为索引的。

简单说来，sender 可以通过 account: signer 来操作自己的账号资源（读写），自己的定义，自己做主；

但想操作其他人的账号资源，必须通过其他人定义的 public fun 方法来操作，而访问 public fun 则通过他人的地址访问（use 关键字）。



# move 和区块链的关系

区块链相当于操作系统；

move 相当于操作系统提供出去的 C 函数。

move 导出 Rust crate 和 区块链交互。

move 代码通过部署到区块链运行在区块链中。



# move_to和 deposit 的区别

move_to 是  move 的原语，即给某个账户资源；

deposit 是一个应用调用，账户创建时会有 balance\<Token\> 资源，deposit 是获得这个资源的可变引用，并更新。



# 在 move 中是怎么防止在代码中直接给账号加 0x1::token 余额（balance\<token\>）的？

因为 0x1::token 是 0x1  这个地址下的资源，只有这个拥有这个地址的账户才能操作这个资源，一般的账户肯定不是 0x1 地址，无法操作这个资源，自然无法随意的增加 0x1::token 余额。

同理，使用一般的账户创建另一个 token ，对这个 token 的读写访问规则，则由这个账户对应的地址下的 module 代码定义决定，外部只能访问他定义的 public fun 方法来操作他定义的 token。

 总之，资源是私有的，token，balance\<token\> 也是资源，想怎么玩，账户地址对应的 module 有自己的规则。其它账户无法随意修改，只能通过 public fun 来交互。



# 是不是只要 create_account_with_address 传入 0x1 地址就可以获得 0x1 账号进而修改他的资源？

话虽如此，但只要不在 0x1::module  下的代码，都必须通过其定义的 public fun 来操作资源，因为想获得资源实体引用（borrow* 方法）必须是 0x1 它自己。还是那句话，不在对应资源地址的 module 下写代码，就只能用 public fun 来操作资源，即使 create_account_with_address 获得这个资源的账号也打破不了这个规则。



# borrow\* 方法只能借用同一个 module 定义下的资源吗？

是的，想 borrow_global\*\<Resuoce\> 成功，前提条件是这个 Resouce 所在 module 和调用 borrow\*代码 是在一起的。相当于 module 只能 borrow 自己定义的 Resource，即使这个 Resource 的实体是在别的 account 下。

由此可见，Resource 一旦定义，任何账号都可以用（move_to），但规则则由定义的 module 决定。可以大胆的想象这么一个恶作剧合约，定一个 token X，然后所有账号都可以用它来交易，到了某个时刻把这些账号下的 token X 全都清零或转移到某个账号下，除了定义这个 token X 账号所有者，使用这个 token X 的其它账号用户没有什么办法去改变 token X 合约，只能承受损失。这么做没什么现实意义，但可以更深刻的理解 Resource，token，account，move_to，deposit， borrow等概念。



# 为什么使用 SMT 访问 key - value 解构而不是直接从 RocksDB 依靠 key 读出 value？

因为 SMT 是 sparce merkle tree，即稀疏默克尔树，有 verfication 的功能，轻节点可以依靠这个能力快速验证 key value 值是否存在于某个 block中，使用 SMT 既有默克尔树的特点（轻节点验证），又利用 RocksDB 实现 O(1) 查找（不考虑 key 碰撞的情况下）（轻节点查询）。