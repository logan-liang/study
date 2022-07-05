---
title: Go-Context
tags:
  - Context
categories:
  - Golang
comments: true
date: 2022-07-05 17:04:25
---


### context的定义

在go语言中用来设置截止日期、同步信号，传递请求相关值得结构体。
其中包括：
* Deadline-返回context.Context被取消的时间，也就是完成工作的截止时间。
* Done-返回一个Channel，这个Channel会在当前工作完成或者上下文被取消后关闭，多次调用Done方法会返回同一个Channel；
* Err-返回context的结束原因，它只会在Done方法对应的Channel关闭时返回非空的值；取消-返回canceld错误；超时-返回DeadlineExceeded错误。
* Value-从context中获取键对应的值，对于同一个上下文来说，多次调用Value并传入相同的Key会返回相同的结果。

### 设计原理

目的：在Goroutine构成的树形结构中对信号进行同步以减少计算资源的浪费。
* 如果创建多个Goroutine来处理一次请求，而context的作用是在不同的Goroutine之间同步请求特定数据、取消信号以及处理请求的截止日期。
* 每一个context都会从最顶层的Goroutine一层一层传递到最下层。context可以在上层Goroutine执行出现错误时，将信号及时同步给下层。

### 相关函数
1、context.WithCancel：返回一个可以手动取消的Context，可以手动调用cancel()方法以取消该context。
2、context.WithDeadline & context.WithTimeout：可以自定义超时时间，时间到了自动取消context。其实withTimeout就是对WithDeadline的一个封装。
3、context.WithValue：可以传递数据的context，携带关键信息，为全链路提供线索。