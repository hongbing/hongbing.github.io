---
layout: post
title: linux系统之block_dump
categories: blog
nav:
  - name: linux
    link: "#linux"
---


当系统的iowait值比较高时，表明系统在处理大量的磁盘操作，我们可以通过`iostat`，`vmstat`命令从宏观上来查看系统的一些io参数，比如某设备的读写速度，cpu的使用情况。

```
#iostat
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          17.62    0.02    3.20    1.37    0.00   77.80

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               1.86        22.79        17.90    4910899    3857428

#vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0 619448 167984   5728 366044   12   13    52    40  125  393 18  3 78  1  0

```
<!-- more -->
但是宏观的信息只能帮助我们得到当前系统的现状，无法了解到具体是哪些进程在频繁执行io操作，以及具体操作的文件是什么。

如果使用的linux内核版本 >= 2.6.20，且内核开启了  CONFIG_TASK_DELAY_ACCT,   CONFIG_TASK_IO_ACCOUNTING,   CONFIG_TASKSTATS   and  CONFIG_VM_EVENT_COUNTERS 选项，python版本在2.5及其以上，我们可以使用`iotop -P`来查看I/O调度器（I/O Scheduler）管理的所有进程，不使用`-P`选项，默认显示所有的线程（注意`iotop`命令需要在root权限下运行）。然后可以使用`lsof -p [pid]`来分析到底该进程正在频繁操作哪些文件。

但是如果使用的系统内核版本 < 2.6.20，需要搞清楚哪些进程在进行大量的io操作，又该如何呢？

可以使用linux内核提供的`block_dump`参数。linux doc对block_dump的描述：

> block_dump
block_dump enables block I/O debugging when set to a nonzero value. More
information on block I/O debugging is in Documentation/laptops/laptop-mode.txt.

下面是laptop-mode.txt中对block_dump的描述：

> If you want to find out which process caused the disk to spin up, you can
gather information by setting the flag /proc/sys/vm/block_dump. When this flag
is set, Linux reports all disk read and write operations that take place, and
all block dirtyings done to files. This makes it possible to debug why a disk
needs to spin up, and to increase battery life even more. The output of
block_dump is written to the kernel output, and it can be retrieved using
"dmesg". When you use block_dump and your kernel logging level also includes
kernel debugging messages, you probably want to turn off klogd, otherwise
the output of block_dump will be logged, causing disk activity that is not
normally there.


从上面的描述可以知道，block_dump是一个调试选项，选项打开的时候，会把当前所有磁盘读写操作的信息输出到内核输出，可以通过`dmesg`来查看输出内容。执行下面的脚本可以在/tmp/diskio.log中看到相关的信息。

{% highlight bash%}
#!/bin/sh
/etc/init.d/klogd stop
# clear dmesg
dmesg -c 2>/dev/null
echo 1 > /proc/sys/vm/block_dump
if [ -f /tmp/diskio.log ]; then
        cat /dev/null > /tmp/diskio.log
fi
#waiting 5 seconds to print io info to kernel output
sleep 5
dmesg -c >> /tmp/diskio.log
echo 0 > /proc/sys/vm/block_dump
/etc/init.d/klogd start
{% endhighlight%}

下面是从/tmp/diskio.log中截取的一部分内容：

 	[88005.820372] jbd2/sda1-8(140): WRITE block 239498592 on sda1 (8 sectors)
	[88005.821425] jbd2/sda1-8(140): WRITE block 239498600 on sda1 (8 sectors)
	[88006.732202] kworker/u8:1(13277): WRITE block 2537400 on sda1 (8 sectors)
	[88006.732327] kworker/u8:1(13277): WRITE block 25003008 on sda1 (16 sectors)
	[88006.732354] kworker/u8:1(13277): WRITE block 1599768 on sda1 (8 sectors)

解释一下输出的内容，[88005.820372]是机器启动到现在的时间，单位为秒， jbd2/sda1-8(140)进程名和pid，后面的内容为磁盘写数据的内容，block后面的数字表示的不是文件层面的block，而是硬件层面的block（也即是扇区），扇区的大小为512B，而大多数文件系统的block大小为4K，因此转换为文件系统的block只需要将后面的值除以8。

通过下面的命令可以统计io最频繁的进程名：

```sh
cat /tmp/diskio.log | awk -F"[() \t]" '/(READ|WRITE|dirtied)/ {activity[$2]++} END {for (x in activity) print x, activity[x]}'| sort -nr -k2
```

结果为：

 	jbd2/sda1-8 21
	kworker/u8:1 7
	rs:main 3
	sh 2

仅仅凭上面的信息是无法知道具体在写哪一个文件。要知道哪个文件是io操作大户，一种方式是前面提到的通过`lsof -p [pid]`命令来分析进程打开的文件，找到正在频繁操作的文件。
另一种方式是通过[debugfs](https://en.wikipedia.org/wiki/Debugfs)工具来分析。

首先根据扇区号，得到文件块号，然后利用debugfs得到inode号，最后通过inode得出文件名。

我们分析kworker的一条记录

 	[88006.732202] kworker/u8:1(13277): WRITE block 2537400 on sda1 (8 sectors)

扇区号：2537400，文件块号：2537400/8=317175

在使用debugfs之前，需要先挂载。

```
$sudo mount -t debugfs none /sys/kernel/debug
mount：none 已挂载或 /sys/kernel/debug 忙
mount：根据 mtab，none 已挂载于 /sys/kernel/debug
```

挂载完成后，就可以调用icheck和ncheck来分别得到inode号和文件名了。

```
$sudo debugfs -R 'open /dev/sda1'
debugfs 1.42.9 (4-Feb-2014)
$sudo debugfs -R 'icheck 317175' /dev/sda1
debugfs 1.42.9 (4-Feb-2014)
Block	Inode number
317175	3545002
$sudo debugfs -R 'ncheck 8' /dev/sda1
debugfs 1.42.9 (4-Feb-2014)
Inode	Pathname
3545002	/home/hongbing/.cache/google-chrome/Default/Cache/data_1
```
![blockdump](/images/linuxofblockdump/blockdump.jpg)

##### 参考
[1] [http://www.vpsee.com/2009/08/monitor-process-io-activity/](http://www.vpsee.com/2009/08/monitor-process-io-activity/)

[2] [http://www.vpsee.com/2010/07/monitoring-process-io-activity-on-linux-with-block_dump/](http://www.vpsee.com/2010/07/monitoring-process-io-activity-on-linux-with-block_dump/)

[3] [https://www.kernel.org/doc/Documentation/laptops/laptop-mode.txt](https://www.kernel.org/doc/Documentation/laptops/laptop-mode.txt)

[4] [http://stackoverflow.com/questions/249570/how-can-i-record-what-process-or-kernel-activity-is-using-the-disk-in-gnu-linux](http://stackoverflow.com/questions/249570/how-can-i-record-what-process-or-kernel-activity-is-using-the-disk-in-gnu-linux)

[5] [http://blog.chinaunix.net/uid-24774106-id-4096470.html](http://blog.chinaunix.net/uid-24774106-id-4096470.html)
