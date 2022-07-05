---
title: 性能调试：分析并优化程序
tags:
  - 性能调优
categories:
  - Golang
comments: true
date: 2021-03-31 11:17:46
---


### 时间和内存消耗

可以用这个便捷脚本xtime来测量：

```
    #!/bin/sh
    /usr/bin/time -f '%Uu %Ss %er %MkB %C' "$@"
```
### 用 go test 调试

我们可以用 gotest 标准的 -cpuprofile 和 -memprofile 标志向指定文件写入 CPU 或 内存使用情况报告

```
    go test -x -v -cpuprofile=prof.out -file x_test.go
```

### 用pprof 调试

你可以在单机程序 progexec 中引入 runtime/pprof 包；这个包以 pprof 可视化工具需要的格式写入运行时报告数据。对于 CPU 性能分析来说你需要添加一些代码：

```
var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to file")

func main() {
	flag.Parse()
	if *cpuprofile != "" {
		f, err := os.Create(*cpuprofile)
		if err != nil {
			log.Fatal(err)
		}
		pprof.StartCPUProfile(f)
		defer pprof.StopCPUProfile()
	}
```
代码定义了一个名为 cpuprofile 的 flag，调用 Go flag 库来解析命令行 flag，如果命令行设置了 cpuprofile flag，则开始 CPU 性能分析并把结果重定向到那个文件。（os.Create 用拿到的名字创建了用来写入分析数据的文件）。这个分析程序最后需要在程序退出之前调用 StopCPUProfile 来刷新挂起的写操作到文件中；我们用 defer 来保证这一切会在 main 返回时触发。

现在用这个 flag 运行程序：`progexec -cpuprofile=progexec.prof`

然后可以像这样用 gopprof 工具：`gopprof progexec progexec.prof`

gopprof 程序是 Google pprofC++ 分析器的一个轻微变种；关于此工具更多的信息，参见https://github.com/gperftools/gperftools。

如果开启了 CPU 性能分析，Go 程序会以大约每秒 100 次的频率阻塞，并记录当前执行的 goroutine 栈上的程序计数器样本。

此工具一些有趣的命令：

* `topN`:用来展示分析结果中最开头的 N 份样本，例如：top5 它会展示在程序运行期间调用最频繁的 5 个函数
* `web 或 web 函数名`:该命令生成一份 SVG 格式的分析数据图表，并在网络浏览器中打开它（还有一个 gv 命令可以生成 PostScript 格式的数据，并在 GhostView 中打开，这个命令需要安装 graphviz）。函数被表示成不同的矩形（被调用越多，矩形越大），箭头指示函数调用链
* `list 函数名 或 weblist 函数名`:展示对应函数名的代码行列表，第 2 列表示当前行执行消耗的时间，这样就很好地指出了运行过程中消耗最大的代码
如果发现函数 runtime.mallocgc（分配内存并执行周期性的垃圾回收）调用频繁，那么是应该进行内存分析的时候了。找出垃圾回收频繁执行的原因，和内存大量分配的根源


```
var memprofile = flag.String("memprofile", "", "write memory profile to this file")
...

CallToFunctionWhichAllocatesLotsOfMemory()
if *memprofile != "" {
	f, err := os.Create(*memprofile)
	if err != nil {
		log.Fatal(err)
	}
	pprof.WriteHeapProfile(f)
	f.Close()
	return
}
```
用 -memprofile flag 运行这个程序：`progexec -memprofile=progexec.mprof`



