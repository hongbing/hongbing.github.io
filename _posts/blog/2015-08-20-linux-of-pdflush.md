---
layout: post
title: linux系统之pdflush
category: blog
tags: linux pdflush
---


当我们向磁盘文件写入数据时，由于内存与磁盘之间数据读取的差异，操作系统不会直接将数据写透（write through）到磁盘上，而是会写到页高速缓存（page cache）中，真正的写磁盘操作会延迟进行，称为写延迟（write delay）。当page cache中的数据比后端存储中的数据还要新时，称page cache中的数据为脏数据（或者脏页，dirty page）。page cache对被缓存的页面范围的定义非常宽泛，任何基于页的对象，包括各种类型<!-- more -->的文件和各种类型的内存映射都可以。如果计算机在运行过程中突然掉电，脏页会来不及写回磁盘，会造成数据丢失。为了尽可能的减少突然掉电带来的数据丢失，同时为了保证系统有足够多的空闲内存空间，操作系统会主动执行将脏页写回磁盘的操作，这也就是pdflush线程的工作。

可以在`/proc/meminfo`中查看系统page cache的信息。

```
	MemTotal:        1943676 kB
	MemFree:          165132 kB
	Buffers:           23784 kB
	Cached:           511048 kB
	SwapCached:            0 kB
	......
	SwapTotal:       1989628 kB
	SwapFree:        1989360 kB
	Dirty:                60 kB
	Writeback:             0 kB
	AnonPages:       1009132 kB
	Mapped:           229812 kB
	Shmem:            195912 kB
	Slab:             163756 kB
	......
```

Cached即表示当前page cache的大小，Dirty表示脏页的大小。

在linux内核2.6，操作系统维护一组内核线程pdflush执行脏页回写，`/proc/sys/vm/nr_pdflush_threads`参数可以设置系统中pdflush的线程数。在2.6.32内核版本，pdflush已经被Flusher线程组所取代，Flusher与pdflush的主要区别在于，**pdflush线程与任何任务无关，它们是面向系统所有磁盘的全局任务**，而Flusher线程组可以针对**每个磁盘独立执行回写操作**。下面我们仍然以pdflush来分析。

在下面3种情况下，pdflush线程会被唤醒执行：

	 当空闲内存低于一个特定的阈值时;
	 当脏页在内存中驻留时间超过一个特定的阈值时;
	 当用户调用sync()和fsync()系统调用时。
	
第一种情况保证当空闲内存过低时，脏页回写释放内存；第二种情况保证脏页不会无限期的驻留在内存中，最后一种是应用层主动要求执行脏页刷新，可以认为前两种情况是被动式的，是在满足空间（内存空闲空间）和时间（脏页驻留时间）任一条件下进行的。最后一种是主动式的。

首先看看被动式的空间阈值，空间阈值包括两个，一个是`/proc/sys/vm/dirty_background_ratio`，另一个是`/proc/sys/vm/dirty_ratio`。

查看Kernel version 2.6.29的doc，看看该doc是怎么描述这两个参数的：

```
==============================================================
dirty_background_ratio

Contains, as a percentage of total available memory that 
contains free pages and reclaimable pages, the number of pages at 
which the background kernel flusher threads will start writing out
dirty data.

The total avaiable memory is not equal to total system memory.

==============================================================
dirty_ratio

Contains, as a percentage of total available memory that 
contains free pages and reclaimable pages, the number of pages at 
which a process which is generating disk writes will itself 
start writing out dirty data.

The total avaiable memory is not equal to total system memory.

==============================================================
```

从doc中可以看出，含义都是表示`占可用内存空间的百分比`。但是区别在达到**dirty_background_ratio**这个阈值后，pdflush线程会`异步`执行脏页回写，而当达到**dirty_ratio**阈值时，当时执行写操作的进程会被强制`同步`执行脏页回写操作，此时所有进程的写操作都会被阻塞,直到脏页占比降低到dirty_ratio之下，如果此时的脏页率仍然在dirty_backgroud_radio之上，将调用pdflush执行异步刷新。

默认的**dirty_ratio**的值应该设置得比**dirty_background_ratio**大。看看工作机中与dirty有关的参数：

```
$ sysctl -a | grep dirty
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 5
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 3000
vm.dirty_ratio = 10
vm.dirty_writeback_centisecs = 500
```

这里有一个需要注意的点，占比依据的是`可用内存`，并不是全部内存。可用内存如何计算？

> MemFree + Cached - Mapped 

因此，按照前面给出的数字，可用内存为435M，那么当脏页达到21.75M时，就会触发空间阈值。

pdflush唤醒后会调用`background_writeout(int)`函数，该函数需要传入一个整形值，表示写回的页数。当下面两个条件满足时，pdflush停止回写操作：

	 指定的最小数目的脏页数被写回到磁盘；
	 空闲内存数已经回升，超过了阈值dirty_background_ratio

只有达到上面两个条件才会停止回写，除非将所有脏页都写回到磁盘，没有剩余的脏页可以回写了。


再来看看被动式的时间阈值`dirty_expire_centisecs`（脏页的过期时间），该值在linux doc中是这样描述的：

```
==============================================================
dirty_expire_centisecs

This tunable is used to define when dirty data is old enough 
to be eligible for writeout by the kernel flusher threads. 

It is expressed in 100 ths of a second.  
Data which has been dirty in-memory for longer than this interval
 will be written out next time a flusher thread wakes up.
==============================================================
```

**dirty_expire_centisecs**表示脏页在内存中驻留的时间阈值，该值的单位为1/100 秒。另外还有一个参数`dirty_writeback_centisecs`

```
==============================================================
dirty_writeback_centisecs

The kernel flusher threads will periodically wake up 
and write old data out to disk.  
This tunable expresses the interval between those wakeups,
in 100 ths of a second.

Setting this to zero disables periodic writeback altogether.
==============================================================
```

表示系统多久唤醒pdflush来执行回写操作。系统启动时会设置一个定时器，时间为百分之dirty_writeback_centisecs秒，当时间到时，唤醒pdflush线程将驻留时间超过百分之dirty_expire_centisecs秒的脏页回写到磁盘，完成后睡眠，然后重置定时器时间为百分之dirty_writeback_centisecs秒等待下一次唤醒，也即周期性的执行脏页回写操作。

最后来看看主动式的回写操作。当调用fsync()时，会传入一个文件描述符fd，fsync会保证该文件的所有脏页都被刷新到磁盘才返回。

另外还有一种情况，是用户态在调用write函数时，当页面被标记为脏后会检查当前的脏页比率，根据条件刷新脏页。此时的刷新策略与脏页占可用内存比率大于dirty_ratio时一致，同步刷新，阻塞其它的write操作，当比率降到dirty_backgroud_ratio后，wakeup pdflush执行异步刷新。因此用户态调用write()函数也可能引发脏页刷新操作。

参数调优应该根据具体的应用场景来确定，可以将参数值添加到`/etc/sysctl.conf`文件中，然后执行`sysctl -p`使其生效。


![OMGLookather](/images/linuxofpdflush/OMGLookather.jpg)


**参考**：
[1] [http://www.westnet.com/~gsmith/content/linux-pdflush.htm](http://www.westnet.com/~gsmith/content/linux-pdflush.htm)
[2] [https://www.kernel.org/doc/](https://www.kernel.org/doc/)
[3] [http://www.oenhan.com/linux-cache-writeback](http://www.oenhan.com/linux-cache-writeback)
