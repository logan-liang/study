---
title: 同步锁、读写锁、WaitGroup
tags:
  - Golang
categories:
  - 锁的应用
comments: true
date: 2021-04-26 14:55:30
---


### 概述

Go 语言中的sync包提供了两种锁类型：sync.Mutex和sync.RMWutex，前者是互斥锁，后者是读写锁。

同时，又提供了一种对主线程等待goroutine执行完毕的方法，sync.WaitGroup。


### 互斥锁

互斥锁是传统的并发程序对共享资源进行访问控制的主要手段，在Go中，似乎更推崇由channel来实现资源共享和通信。它由标准库代码包sync中的Mutex结构体类型代表。只有两个公开的方法：

```
    Lock() //获得锁
    UnLock() //释放锁
```

* `Lock()`：使用加锁后，不能再继续对其加锁（同一个goroutine中，即：同步调用），否则会panic。只有在unlock 之后才能再次lock。异步调用Lock()，是正当的锁竞争，当然不会有panic了。适用于读写不确定场景，即读写次数没有明显的却别，并且只允许只有一个读或者写的场景，所以该锁也叫做全局锁。

* `UnLock()`：用于解锁，如果在使用Unlock前未加锁，就会引起一个运行错误。已经锁定的Mutex并不与特定的goroutine相关联，这样可以利用一个goroutine对其加锁，再利用其他goroutine对其解锁。

建议：同一个互斥锁的成对锁定和解锁操作放在同一层次的代码块中。

```
var lck sync.Mutex
func foo() {
    lck.Lock() 
    defer lck.Unlock()
    // ...
}
```

lck.Lock() 会阻塞知道获取锁，然后利用defer语句在函数返回时自动释放锁。

### 读写锁

读写锁是分别针对读操作和写操作进行锁定和解锁操作的互斥锁。在Go语言中，读写锁由结构体类型sync.RWMutex代表。

基本遵循原则：

* 写锁定情况下，对读写锁进行读锁定或者写锁定，都将阻塞；而且读锁与写锁之间是互斥的。
* 读锁定情况下，对读写锁进行锁定，将阻塞；加读锁时不会阻塞；
* 对未被写锁定的读写锁进行写解锁，会引发Panic；
* 对未被读锁定的读写锁进行读解锁的时候也会引发Panic；
* 写解锁在进行的同时会试图唤醒所有因进行读锁定而被阻塞的goroutine；
* 读解锁在进行的时候会试图唤醒一个因进行写锁定而被阻塞的goroutine；
* 与互斥锁类似，sync.RWMutex类型的零值就已经是立即可用的读写锁了。在此类型的方法集合中包含了两队方法。即：

```
func (*RWMutex) Lock // 写锁定
func (*RWMutex) Unlock // 写解锁
func (*RWMutex) RLock // 读锁定
func (*RWMutex) RUnlock // 读解锁
```