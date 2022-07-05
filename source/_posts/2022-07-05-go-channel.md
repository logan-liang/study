---
title: Go-Channel
tags:
  - Channel
categories:
  - Golang
comments: true
date: 2022-07-05 17:04:18
---


### Channel数据结构

```
type hchan struct {
  //channel分为无缓冲和有缓冲两种。
  //对于有缓冲的channel存储数据，借助的是如下循环数组的结构
	qcount   uint           // 循环数组中的元素数量
	dataqsiz uint           // 循环数组的长度
	buf      unsafe.Pointer // 指向底层循环数组的指针
	elemsize uint16 //能够收发元素的大小
  
	closed   uint32   //channel是否关闭的标志
	elemtype *_type //channel中的元素类型
  
  //有缓冲channel内的缓冲数组会被作为一个“环型”来使用。
  //当下标超过数组容量后会回到第一个位置，所以需要有两个字段记录当前读和写的下标位置
	sendx    uint   // 下一次发送数据的下标位置
	recvx    uint   // 下一次读取数据的下标位置
  
  //当循环数组中没有数据时，收到了接收请求，那么接收数据的变量地址将会写入读等待队列
  //当循环数组中数据已满时，收到了发送请求，那么发送数据的变量地址将写入写等待队列
	recvq    waitq  // 读等待队列
	sendq    waitq  // 写等待队列

	lock mutex //互斥锁，保证读写channel时不存在并发竞争问题
}
```

### 过程详解

#### 先入先出

目前的Channel收发操作均遵循了先进先出的设计，具体规则如下：

* 先从Channel读取数据的Goroutine会先接收到数据；
* 先向Channel发送数据的Goroutine会得到先发送数据的权利；

#### 发送步骤：

* 锁定整个hchan通道结构。
* 确定发送，recvq不为空，将元素直接写入goroutine同时挂起，等待被唤醒；recvq为空，则确定缓冲区是否可用，如果可用，则把数据写到缓冲区中；如果缓冲区满了，则把当前数据写入goroutine，并且当前goroutine保存在sendq中，进入休眠，等待被唤醒。
* 写入完成释放锁。

#### 读取步骤：

* 先获取hchan全局锁。
* 尝试从sendq队列中获取数据。
* 如果sendq不为空且没有缓冲区，取出g并读取数据，然后唤醒g，结束读取；
* 如果sendq为空且有缓冲区，从缓冲区队列获取数据，再从sendq取出g，将g中的数据存放到buf队列尾；
* 如果sendq不为空且有缓冲区，直接读取缓冲区数据；
* 如果sendq为空且没有缓冲区，将当前goroutine加入到sendq队列，进入睡眠，等待被写入goroutine唤醒。
* 结束读取释放锁。