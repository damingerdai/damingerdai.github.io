---
title: 2026年5月8号面试总结
date: 2026-05-08 19:56:55
tags: [高级前端工程师, 面试]
categories: [前端]
---

# 前言

2026年5月8号有一场面试，是对方架构师来面试的我。总的来说，面试的效果不是很好。
面试的岗位是前端岗位，但是问的很多是后端架构的设计，我认为我的准备不是很好，我需要继续努力补全自己。

# 问题

## Java的虚拟线程

问题：Java的虚拟线程是什么，怎么创建？

回答如下：

在Java 21中，引入了虚拟线程（Virtual Threads）来简化和增强并发性，这使得在Java中编程并发程序更容易、更高效。

虚拟线程，也称为“用户模式线程（user-mode threads）”或“纤程（fibers）”。该功能旨在简化并发编程并提供更好的可扩展性。虚拟线程是轻量级的，这意味着它们可以比传统线程创建更多数量，并且开销要少得多。这使得在自己的线程中运行单独任务或请求变得更加实用，即使在高吞吐量的程序中也是如此。

在Java 21中创建和使用虚拟线程有多种方法：

### 1. 使用静态构建器方法

Thread.startVirtualThread方法将可运行对象作为参数来创建，并立即启动虚拟线程，具体如下代码：

```java
Runnable runnable = () -> {
    System.out.println("Hello, www.didispace.com");
};

// 使用静态构建器方法
Thread virtualThread = Thread.startVirtualThread(runnable);
```

也可以使用Thread.ofVirtual()来创建，这里还可以设置一些属性，比如：线程名称。具体如下代码：

```java
Thread.ofVirtual()
        .name("didispace-virtual-thread")
        .start(runnable);
```

### 2. 与ExecutorService结合使用

从Java 5开始，就推荐开发人员使用ExecutorServices而不是直接使用Thread类了。现在，Java 21中引入了使用虚拟线程，所以也有了新的ExecutorService来适配，看看下面的例子：

```java
Runnable runnable = () -> {
    System.out.println("Hello, www.didispace.com");
};

try (ExecutorService executorService = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100; i++) {
        executorService.submit(runnable);
    }
}
```

### 3. 使用虚拟线程工厂

开发者还可以创建一个生成虚拟线程的工厂来管理，具体看下面的例子例子：

```java
Runnable runnable = () -> {
    System.out.println("Hello, www.didispace.com");
};

ThreadFactory virtualThreadFactory = Thread.ofVirtual()
        .name("didispace", 0)
        .factory();

Thread factoryThread = virtualThreadFactory.newThread(runnable);
factoryThread.start;
```

对于虚拟线程的概念，你只需要有一个基本的认识。

- 虚拟线程是由JVM管理的轻量级线程。
- 虚拟线程不需要任何显式分配或调度。
- 虚拟线程非常适合I/O密集型任务或需要大量并行性的任务。
- 虚拟线程也可以用来实现异步操作。

## Golang的cmp模型

GMP 是 Go 运行时（Runtime）层面的调度模型，它实现了用户态线程（Goroutine）与内核态线程（M）之间的多对多复用，是 Go 能够支持高并发的核心原因

- G (Goroutine): 协程。它是待执行的任务，存储了协程的栈、状态及指向任务函数的指针。G 的体积很小（初始仅 2KB），由 Go 运行时管理。
- M (Machine): 操作系统的物理线程。协程最终必须绑定到 M 上才能在 CPU 上运行。M 并不保存状态，它只是一个执行体。
- P (Processor): 逻辑处理器。它是 G 和 M 之间的“中介”或“本地资源池”。P 维护着一个本地就绪队列（Local Runqueue）。M 必须获取到 P 才能执行 G。
- 在早期（Go 1.1 之前）只有 GM 模型，由于所有 M 都要抢夺全局队列的锁来获取 G，导致严重的锁竞争。引入 P 后，每个 P 都有本地队列，减少了锁竞争，并实现了 Work Stealing 和 Hand Off 机制，大大提升了效率

> gemini帮我回答的

## MySQL 事务隔离级别

SQL:1992 标准定义了四种事务隔离级别，分别是读未提交（Read Uncommitted）、读已提交（Read Committed）、可重复读（Repeatable Read）和串行化（Serializable）。不同的隔离级别下会产生脏读、幻读、不可重复读等相关问题，因此在选择隔离级别的时候要根据应用场景来决定，使用合适的隔离级别。
MySQL 中的 InnoDB 存储引擎提供了 SQL 标准所描述的四种事务隔离级别，默认事务隔离级别是可重复读（Repeatable Read）。

- 读未提交（Read Uncommitted）允许脏读，就是在该隔离级别下，可能读到其他会话未提交事务修改的数据，存在脏读、不可重读读、幻读的问题。
- 读已提交（Read Committed）只能查询到已提交的数据。这是 Oracle 数据库默认的事务隔离级别。存在不可重读读、幻读的问题。
- 可重复读（Repeatable Read）就是在一个事务里相同条件下，无论何时查到的数据都和第一次查到的数据一致。这是 MySQL 数据库 InnoDB 引擎默认的事务隔离级别。在范围查询时存在幻读的问题。
- 串行化（Serializable）是最高的事务隔离级别，它严格服从 ACID 特性的隔离级别。所有的事务依次逐个执行，事务之间互不干扰，该级别可以防止脏读、不可重复读以及幻读。但每个事务读数据时都需要获取表级的共享锁，导致读和写都会阻塞，性能极低。


# 参考资料

- [Java 21 新特性：虚拟线程（Virtual Threads）](https://www.cnblogs.com/didispace/p/17735173.html)
- [MySQL | 事务隔离级别详解](https://juejin.cn/post/7136112451959848991)