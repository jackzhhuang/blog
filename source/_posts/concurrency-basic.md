---
title: concurrency_basic
date: 2023-04-16 18:46:39
published: false
tags:
categories:
- Concurrency

---

## 前置概念

### 并发与并行

并发，英文 concurrency，直译就是同时发生。并行，parallelism，直译就是平行。所以虽然一字之差，意思却是不同的，但也很好理解，前者是同时发生，后者是一起做事情。

最简单的比喻就是肯德基点餐场景——几个顾客同时点餐，服务员们一个准备汉堡，一个准备咖啡。顾客同时点餐就是并发，服务员们分工去满足顾客就是并行。

有了清楚的认识并行并发概念后，在处理异步问题的时，应该考虑，我们是解决了并行问题，还是解决了并发问题亦或都能解决？

当然，为了不啰嗦或者咬文嚼字，偶尔我们可能会只用“并发”或者“并行”来代表“并发和并行”两个概念。



### 从大到小的并发场景

#### 位级并行

即 8 位或者 16 位等基本指令位数，例如相对于 8 位的 CUP，64 位的CPU 可以一次并行处理 8 个 8 位的数字。



#### 指令集并行

CPU 执行指令的时候实际上会预先判断后续分支，先把可能的分支给执行了：

```c++
int number = 0;
bool is_set = false;

void set_number() {
  number = 99;
  is_set = true;
}

void check_number() {
  if (is_set) {
    print(number);
  }
}

start_thread(set_number);
start_thread(check_number);
```

 上述伪代码中，check_number 有可能优先执行，且因为 is_set 会被设置为 true，因此 CPU 会预判需要打印 number，因此，有可能会输出：

```c++
0
```

这就是指令集的并行优化。



#### 任务级并行

即每个 CPU 或者每台服务器并行处理。

