---
layout: darft
title: Channel
tags:
  - Golang
categories:
  - 信道
comments: true
date: 2021-05-06 09:14:49
---


## 介绍

作为Go核心的数据结构和Goroutine之间的通信方式，Channel是支撑Go语言高性能并发编程模型的重要结构。

### 设计原理

Go语言中最常见的、也是经常被人提及的设计模式就是：不要通过共享内存的方式进行通信，而是应该通过通信的方式共享内存。

虽然我们在Go语言中也能使用共享内存加互斥锁进行通信，但是Go语言提供了一种不同的并发模型，即通信顺序进程（CSP）。Goroutine和Channel分别对应CSP中的实体和传递信息的媒介，Goroutine之间会通过Channel传递数据。

#### 先入先出

目前的Channel收发操作均遵循了先进先出的设计，具体规则如下：

* 先从Channel读取数据的Goroutine会先接收到数据；
* 先向Channel发送数据的Goroutine会得到先发送数据的权利；

#### 无锁管道

锁是一种常见的并发控制技术，我们一般会将锁分成乐观锁和悲观锁，即乐观并发控制和悲观并发控制。

* 同步Channel：不需要缓冲区，发送方会直接将数据交给接收方；
* 异步Channel：基于环形缓存的传统生产者消费者模型；
* `chan struct{}`类型的异步Channel：`struct{}`类型不占用内存空间，不需要实现缓冲区和直接发送的语义；

### 数据结构

Go语言的Channel在运行时使用`runtime.hchan`结构体表示。

```
type hchan struct {
	qcount   uint   //channel中的元素个数
	dataqsiz uint   //channel中的循环队列的长度
	buf      unsafe.Pointer  //channel缓冲区数据指针
	elemsize uint16 //表示当前channel能够收发的元素大小
	closed   uint32
	elemtype *_type  //表示当前channel能够收发的元素类型
	sendx    uint   //channel的发送操作处理到的位置
	recvx    uint   //channel的接收操作处理到的位置
	recvq    waitq  //
	sendq    waitq

	lock mutex
}
```

#### 创建管道

```
ch:=make(chan type) //无缓冲
ch:=make(chan type,num) //有缓冲
```

#### 发送与接收

```
ch <- 1 //发送数据
a:=<-ch    //接收数据
```

#### 关闭管道

```
close(ch)
```


