---
layout: post
title: 一次oom的排查过程
description: 一次oom的排查过程
categories: posts blog
---

最近对平台内部的消息中间件添加了从kafka读取数据的功能（平台内部存在着多个消息中间件，主要是由不同的使用场景与历史沿革造成的）。原有逻辑是从mcq中读取消息，现在需要新增从kafka读取消息。<!-- more -->

代码编写完毕后，在测试的过程中发现只要将读取kafka数据的功能开关打开，不一会儿就出现服务卡死，既无法从kafka，也无法从mcq读取数据，偶尔有那么几条数据的读取。同时在日志目录生成了`memory.hprof`的文件。这个文件的生成原因是由于java进程启动参数里面有如下的配置：

> `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=../logs/memory.hprof`

上面的配置表明当进程出现OutOfMemory Error的时候将进程的堆情况（内存情况）dump到logs目录下的memory.hprof文件中。这个文件的大小只有25M，（后来在没有启动读取kafka功能的其他机器上dump了当前进程的堆使用情况，dump下来的文件有1G大小），由于通过top 可以看到正常运行的进程使用的RES内存大小在2G左右，所以这25M的故障时的堆数据可能不能完全表明当时进程堆所处的情况。我们先不看memory.hprof里的内容。
除了依赖进程自身生成堆存储快照，还可以使用`jmap`命令生成堆存储快照：

> `jmap -dump:format=b,file=heap.hprof [pid]`

从生成memory.hprof这个文件，我们得到一个信息，就是进程OOM了，那么到底是什么样的内存溢出了呢？是堆内存还是堆外内存溢出了呢？毕竟上面两种情况溢出都可以叫做`OutOfMemoryError`。可以使用`jstat -gcutil [pid]`查看当前进程的堆使用情况，如下：
![jstat-gcutil](/images/oomproblem/jstat-gcutil.png)

除了gc情况，还可以使用jstat命令查看各个区的容量大小。


+ 查看永久代（perm gen）容量：
![jstat-permcapacity](/images/oomproblem/jstat-permcapacity.png)
这里显示Perm区的容量为40960K.

+ 查看新生代（new）使用的容量：
![jstat-newcapacity](/images/oomproblem/jstat-newcapacity.png)

+ 查看老年代（old）使用容量：
![jstat-oomcapacity](/images/oomproblem/jstat-oldcapacity.png)


另外，在OOM之前一定存在大量的`Full GC`,在java进程启动参数里面还有这样的配置：

> `-XX:+PrintGCDetails -XX:+PrintGCDateStamps` 
> `-Xloggc:../gclogs/gc.log.$nowday`

变量`$nowday`表示当前的时间：

{% highlight bash %}
 nowday=`date +%Y%m%d_%H%M%S`
{% endhighlight %}

上面的配置让进程的GC情况都一一记录到gc.log文件中，我们还可以通过这些日志来查看gc的情况。

```
2016-04-14T17:29:33.351+0800: 257.802: [Full GC 257.802: [CMS: 14273K->14754K(3145728K), 0.2970110 secs] 716960K->14754K(4044544K), [CMS Perm : 40959K->40957K(40960K)], 0.2973540 secs] [Times: user=0.30 sys=0.00, real=0.30 secs]
2016-04-14T17:29:33.650+0800: 258.101: [Full GC 258.101: [CMS: 14754K->14615K(3145728K), 0.1683430 secs] 15839K->14615K(4044544K), [CMS Perm : 40959K->40959K(40960K)], 0.1685680 secs] [Times: user=0.17 sys=0.00, real=0.17 secs]
2016-04-14T17:29:33.819+0800: 258.270: [Full GC 258.270: [CMS: 14615K->13871K(3145728K), 0.1910210 secs] 14770K->13871K(4044544K), [CMS Perm : 40959K->40894K(40960K)], 0.1912510 secs] [Times: user=0.20 sys=0.00, real=0.19 secs]
```
通过上面的gc日志可以看到JVM正在执行FullGC，使用的是CMS收集器，堆内存容量为4044544K，Perm区容量为40960K，且经过GC过后仍然占用40894K，可见是Perm区溢出导致Full GC，而Perm区存储的是class相关的信息。当读取kafka的功能开关打开之后，会加载与kafka client相关的class信息，由于class信息的增多导致Perm区溢出了，Perm溢出一般的解决方法就是增加Perm区大小。

查看进程启动的时候JVM相关配置参数：

```
-Xms4g -Xmx4g  -Xmn1g -XX:PermSize=40m -XX:MaxPermSize=40m -XX:+UseConcMarkSweepGC 
-XX:CMSInitiatingOccupancyFraction=80 -XX:+PrintGCDetails -XX:+PrintGCDateStamps 
-Xloggc:../gclogs/gc.log.$nowday -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=../logs/memory.hprof  -server
```

PermSize最大为40M，与上面GC日志以及jstat看到的内容一致，Perm区设置的比较小，将其改为如下：

> `-Xmn1g -XX:PermSize=64m -XX:MaxPermSize=128m`

由于堆内内存只占用了50%左右，不用更改其大小。

如果是堆内内存溢出，我们还需要通过Memory Analysis，VirtualVm或者Eclipse MAT分析导致溢出的原因，是什么类占据了过多的空间，通过修复代码的方式来解决堆内内存溢出的问题。

![oom](/images/oomproblem/oom.jpg)

