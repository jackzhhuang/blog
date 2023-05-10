---
title: 并发开发模型 —— 基本并发模型
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

我们先从小到大的看一下我们都会遇到什么样的并行场景。

#### 位级并行

即 8 位或者 16 位等基本指令位数，例如相对于 8 位的 CUP，64 位的CPU 可以一次并行处理 8 个 8 位的数字。



#### 指令级并行

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

CPU，cache 和内存的并行：

![CPU, cache 和 内存](https://www.jackhuang.cc/svg/concurrency-memeory-cpu.drawio.svg)



网络各台服务器其实也是一个特大型计算机，它们的结构和计算机内部有点相似：

![网络的并行任务](https://www.jackhuang.cc/svg/concurrency-network-task.svg)



### 学习的过程应该考虑什么问题

市面上五花八门的并行（并发）模型，几乎每个人都有他们看家秘诀去解决各种并行并发问题，但软件没有银弹，没有灵丹妙药，因此，过去，现在和将来，无论遇到什么并发模型，都应该谨慎而不是直接拥抱，每一种模型都一定是一种取舍方案，我们获得了什么好处，由此带来什么问题都应该考虑到，因此，我们学习完一个并发模型，都应该考虑：

1、这个模型解决了并发还是并行还是都解决了？

2、这个模型属于上述三级的哪个架构？

3、这个模型是否有利于写出容错性强货解决分布式问题的代码？



## 线程与锁

### 只有一把锁的情况

我们最常见的就是使用一把锁来控制资源的访问，例如：

```c++
int number = 0;
void incr_number() {
  ++number;
}

int get_number() {
  return number;
}

void loop_incr_number() {
  while (get_number() < 100) {
    incr_number();
  }
}

void check_large_number() {
  while (get_number() >= 100) {
    print("it is large than 100");
    break;
  }
}

start_thread(loop_incr_number());
start_thread(check_large_number());
```

我们的本意是，一个线程增加 number，另一个线程看 number 是否大于等 100，满足条件后，第一个线程退出，第二个线程打印 "it is large than 100" 后也退出。

目前看似乎这么写没什么问题。但有人会问：为什么第 11 行和第 17 行要写成不等式而不是等式呢？

这就是前面说的，我们在处理多线程问题的时候，需要做到易于写出难以写错的代码，即使写错了，也要让错误尽早的暴露出来。这里，使用不等号就是为了防止当有多个线程调用 incr_number 时，while (get_number() != 100) 将有可能进入死循。

因为假设 A 和 B 线程都调用 loop_incr_number，它们都通过 while (get_number() != 100) 的检查，此时假如 number 为 99，那么，A 和 B 都要去调用 incr_number，最后将可能导致 number 为 101。此时，while (get_number() != 100) 不可能再会满足，AB 线程都要进入死循环。

是的，我们的代码对使用方没有任何限制，使用方可能会用多个线程来调用我们的 loop_incr_number，我们作为 loop_incr_number 的设计方，应该有所设计，而不能仅仅只是在文档上写出 —— 这个函数不允许多个线程调用（虽然我知道早期 C 函数有这样的做法，但这个不是挡箭牌。）

那么，是不是使用不等号就没问题了呢？也未必。

试想，如果我们每次增加 number，就调用某个外部函数，如果我们的期望是调用 100 次，多一次或者少一次都会造成致命错误，那么由于我们无法控制使用方有多少个线程在调用我们的 loop_incr_number，若使用方使用 3 个线程去调用 loop_incr_number，则由于线程 cache 的原因，可能会造成 do_something 被调用多于 100 次，如下代码所示：

```c++
void loop_incr_number() {
  while (get_number() < 100) {
    incr_number();
    do_something(); // 每次循环调用一次，一共调用 100 次
  }
}

start_thread(loop_incr_number());
start_thread(loop_incr_number());
start_thread(loop_incr_number());
```

如何做到多个线程调用 loop_incr_number 且不会受到线程 cache 的影响呢？

一个简单的方法就是给 incr_number 函数增加一个互斥锁，这样，即使外部线程再多，由于有互斥锁，这样即使有多个线程在调用 loop_incr_number 也不会出现因为线程 cache 的原因而多于 100 次的结果：

```c++
void syncronized loop_incr_number() {
  while (get_number() < 100) {
    incr_number();
    do_something(); // 每次循环调用一次，一共调用 100 次
  }
}
```

注意以上是伪代码，Java 中使用 sysncronized，C++ Windows 平台上则可以建立关键区从而做到函数级的互斥。这样，外部不管多少个线程调用 loop_incr_number 都会因为这个关键区同步运行。

