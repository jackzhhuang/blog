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