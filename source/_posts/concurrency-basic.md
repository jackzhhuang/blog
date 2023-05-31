---
title: 并发开发模型
date: 2023-04-16 18:46:39
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



<!--more-->



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

 上述伪代码中，check_number 有可能优先执行，且因为 is_set 和 number 没有必然的逻辑联系， CPU 可能会异步执行第 5 和第 6 行代码，即可能先执行 *is_set = true* ，因此 CPU 会先看到 is_set 是 true 了，然后打印 number 值，因此，有可能会输出：

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



## 锁与线程

### 简单锁

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

```c++
void incr_number() {
  ++number;
}
```

上面的代码中，number 可能因为线程 cache 的原因，导致若有两个线程都在跑 incr_number 函数的话，可能前一个函数的结果被后一个函数的结果覆盖的情况。细节是，每一个线程都有自己的 cache，若第一个线程 number 的 cache 是 10，第二个也是 10，第一个线程执行 ++，变为 11，第二个也因为是 10 执行 ++，结果也是 11，那么最后虽然两个线程执行了两次 ++，但结果却是 11。

为了告诉线程， number 是有多线程访问的，不要使用 cache 优化，需要给 incr_number 函数加锁：

```c++
void synchronized incr_number() {
  ++number;
}
```

看上去对写操作进行加锁操作就可以了，但若读操作若不加锁，也是会被 cache 影响，所以，即使是读操作，也应该加上同步锁：

```c++
int synchronized get_number() {
  return number;
}
```

 这时，loop_incr_number 和 check_large_number 应该都不会有问题了，但使用 synchronized 这样的基于代码的锁是非常有局限性的，因为它不能主动解锁，实际上最好使用 mutex 这样的：

```c++
void incr_number() {
  mutex.lock();
  ++number;
}

int get_number() {
  mutex.lock();
  return number;
}
```

上面的 mutex 用 RAII 的管理锁方式去加锁，可以主动释放锁。使用 mutex 的好处是可以探测是否会被阻塞，从而减少了死锁的风险：

```c++
bool incr_number() {
  Lock lock = mutex.trylock();
  if (!lock) {
    return false;
  }
  ++number;
}

int get_number() {
  Lock lock = mutex.trylock();
  if (!lock) {
    return -1;
  }
  return number;
}

start_thread(incr_number);
start_thread(get_number);
```

使用 trylock 可以给我们机会探测是否加锁成功。 相比最开始的 synchronize 就灵活很多。

### 两个锁引起的死锁

一般来说，一个资源就需要一把锁，若两个资源，那么就需要两把锁来控制对它们的访问：

```c++
bool is_set = false;
int number = 10;

void check() {
  mutex_number.lock();
  number++;
  mutex_is_set.lock();
  is_set = true;
}

void print() {
  mutex_is_set.lock();
  if (is_set) {
    mutex_number.lock();
    print(number)
  }
}
start_thread(check);
start_thread(print);
```

上面的伪代码中，严格遵循了一个资源一把锁的原则，这当然是好的，但 check 和 print 加锁顺序正好相法，若两个线程各跑一个函数，极有可能造成死锁：check 线程获得了 mutex_number 锁，等 mutex_is_set，而 print 线程获得了 mutex_is_set 锁，等 mutex_number 锁，两边互等。

当然可以用 trylock 的方式避免，但这样相当于加大了失败的概率，有一定的损耗。

### 好的并发编程建议

以上算是简单的过了一遍使用简单锁和原始线程函数来解决问题的流程。可以看到，在实践编程中，若使用原始的锁和线程，往往会滋生很多潜在的危险：

1、加锁没有考虑全面，造成出现由于 cache 优化的机制而有幻读的风险；

2、使用 synchronize 这样的锁缺少灵活性；

3、两个线程加锁顺序的不一致造成死锁。

因此，一般情况下，尤其是应对复杂编程上下文环境时，不建议直接使用语言提供的原生 API 接口进行加锁和生成线程。应该使用有保障，封装优秀的并发编程库。

此外，跑线程加锁这样的事情应该有章法可循。这里引入最经典最简单的模型：生产者和消费者模型。一方面标准化并发模型，另一方面杜绝了以上讨论到的坑点：

```c++
int number = 10;
void check() {
  producer.lock();
  nunmber++;
  producer.send(number);
}

void print() {
  consumer.lock();
  int number = consumer.receive();
  print(number);
}
```

可以看到，原来简单使用 mutex 来做同步控制，现在使用了生产者-消费者（producer- consumer）来同步，其和原来最本质区别是，number 这个对象不再是两个函数共享，而是彻底解藕。这种编程模型相比使用原生的锁和线程来控制，更易于理解，维护，当然也就容易避免了潜在的 cache 优化和死锁问题。

当然，不管怎么样，异步编程都是有潜在的死锁问题的。生产者消费者模型并不是灵丹妙药，我们还是要注意逻辑上的死锁问题。
