---
layout:     post
title: 分布式ID生成器
category: project
description: ID最大的特点就是唯一，一个ID存在的目的就是唯一性的标识某一个物体，如果唯一性做不到，那么ID也就失去了本质的作用。
---

###1. 什么是ID

ID最大的特点就是**唯一**，一个ID存在的目的就是唯一性的标识某一个物体，如果唯一性做不到，那么ID也就失去了本质的作用。
另外ID还需要保持其它特性：

+ **有序**   对于某些事物需要依靠ID来进行比较，同时对于排序查找也方便。
+ **有意义**  ID也需要包含有意义的信息，比如时间信息,业务信息。
+ **分布式** ID的生成也许并不在一台服务器上，可由多台服务器互不干扰的生成ID。如果采用中心式的ID生成，多半会涉及到锁，而锁意味着成本和性能的下降。
+ **紧凑** 为了性能或者长度的原因，要求ID要保持紧凑。越长的ID可能表达的数字范围越大，但是在存储和网络中要占据更多的空间。因此，设计ID的时候往往是保证在未来的一段时间可用的前提下（查看[2038](https://en.wikipedia.org/wiki/Year_2038_problem)问题），尽可能的缩短ID的长度。`总之，掌握好长度与时间范围之间的trade-off`。

另外，参考原新浪weibo通讯技术专家 宇鹏的[业务系统需要怎样的全局唯一ID](http://weibo.com/p/1001603800404851831206?from=page_100505_profile&wvr=6&mod=wenzhangmod)博文,文中还提到了另外两个特点：

+ **可反解**
反解ID可以为排错提供良好的线索，ID从什么时候生成，在哪个机房生成的，等等情况都可一了解到。
+ **可制造**
可制造针对的是当业务系统失败了，后续需要恢复数据重新进入业务系统，依赖于时间的数据需要在将来的时间段产生过去的时间数据。


###2. 分布式ID生成的难点
分布式ID设计的难点在于对时间的控制，计算机时钟存在着“**时钟偏移**”的情况（查看[分布式环境下的共时性问题,ThereIsNoNow](queue.acm.org/detail.cfm?id=2745385)）。[NTP](http://www.ntp.org/)就是专门针对时钟偏移而设计来同步网络中各服务器时钟的协议。为了保持时间一致，NTP允许服务器的时间倒退。在分布式环境下，如果ID的产生绝对的依赖于时间，应该将NTP的这一特性关闭以免产生重复的ID（或者的ID生成器代码里应该对时间倒退的情况进行判断）。由于网络中各服务器到NTP服务器的跳数可能不同，同时由于网络的抖动，因此难以使得服务器之间的时钟绝对一致。

换个思路，在分布式环境下的时钟不能保证绝对的一致，因此完全利用时钟来保证ID的唯一性是不可能的，但是可以把时间作为ID唯一性保证的一部分，另外利用其他的特性来保证ID全局的唯一性。ID结构中代表时间的那一部分可作排序之用，在ID尾部使用序列号来保证唯一性。同时，在ID结构中间还可以添加一些其他信息，比如机房号，服务器号等等信息。

谈及时间，进一步，到底是精确到秒，还是毫秒。**生成ID的时间维度决定了业务的峰值速度**。由于峰值的时间毕竟比较短，因此，针对峰值在固定长度的ID结构中，设计每秒100万比每毫秒1000更具有灵活性。


###3. 竞品分析

####1）[SnowFlake](http://engineering.twitter.com/2010/06/announcing-snowflake.html)
SnowFlake是为了解决Twitter在分布式环境下的全局ID的问题。
生成的ID需要满足两个要求：

+ **数据量**
每秒能产生数十万个ID，来标识不同的推文
+ **K-Sorted**
ID大致有序。

SnowFlake中的ID长度都是64位，由时间戳、节点号和序列编号组成。

**时间戳**以毫秒为单位，占41位，记录的是从 1288834974657 (Thu, 04 Nov 2010 01:42:54 GMT) 这一时刻到当前时间所经过的毫秒数。有时间戳，必然会存在着一个时间原点，默认情况下的时间元年为1970年1月1日 00:00:01。而在系统设计时，可以以当前的年份来表示元年，这样可以增加ID的使用年限。</br>
**节点号**在源码中被分成两部分，数据中心的 ID 和节点 ID，各自占 5 位。</br>
**序列编号**为本地编号，占12位，从0开始，到4095，然后循环又从0开始，也就是说每一毫秒每个节点能够产生4096的ID号。</br>
还有一位为**保留位**，始终为0。

####2）[Icicle](http://engineering.intenthq.com/2015/03/icicle-distributed-id-generation-with-redis-lua/)

Icicle的ID结构图：

![icicle ID](/images/idgenerator/snowflake.png)

Icicle的ID设计思路来源于SnowFlake，因此与SnowFlake的ID结构有很大的相似性。

**Sign**
第一位为符号保留位，不使用，文中给出的解释为

> which we chose not to use because some platforms make it difficult to get at and it messes with ordering

**Time:**
时间占41位。

**Shard ID:**
由于Icicle是部署在redis上的，redis不支持分布式模式，因此shard id用来标记每一个redis实例，可支持1024个实例。

**Sequence ID:** 与SnowFlake类似。

Icicle支持每一毫秒产生2^10 * 2^12个ID号。

####3）weibo
微博使用了秒级的时间，用了30bit，Sequence 用了15位，理论上可以搞定3.2w/s的速度。用4bit来区分IDC，也就是可以支持16个 IDC，对于核心机房来说够了。剩下的有2bit 用来区分业务，由于当前发号服务是机房中心式的，1bit 来区分热备。没有用满64bit。

####4）Ticktick
使用了30bit 的秒级时间，20bit 给Sequence。这里是有个考虑，第一版实现还是希望到毫秒级，所以20bit 的前10bit给了毫秒来用，剩下10bit给 Sequence。等到峰值提高的时候可以暂时回到秒级。

高位留了2bit 做 Version，或者到时候改造使用更长字节数，用第一位来标识不同 ID，或者可以把这2bit 挪给时间用，可以给系统改造留出一定的时间。

剩下的10bit 留给 MachineID，也就是说当前 ID 生成可以直接内嵌在业务服务中，最多支持千级别的服务器数量。最后有2bit 做Tag 用，可能区分群消息和单聊消息。同时你也看出，这个 ID 最多支持一天10亿消息（原文如此，一天的消息量按照第一版算应该是86400 * 1000 * 2^10 * 2^10 ），也是怕系统增速太快，这2bit 可以挪给 Sequence，可以支持40亿级别消息量，或者结合前面的版本支持到百亿级别。

*参考文献*：

[1][icicle-distributed-id-generation-with-redis-lua](http://engineering.intenthq.com/2015/03/icicle-distributed-id-generation-with-redis-lua/)

[2][Snowflake](https://github.com/twitter/snowflake)

[3][http://www.dengchuanhua.com/132.html](http://www.dengchuanhua.com/132.html)

[4][业务系统需要怎样的全局唯一ID](http://weibo.com/p/1001603800404851831206?from=page_100505_profile&wvr=6&mod=wenzhangmod)
