---
layout: post
title: 硬件中断均衡
description: 硬件与cpu沟通有两种方式，一种是中断，另一种是轮询。中断是硬件主动，产生中断信号，中断控制器将信号传递给cpu，cpu停下手中的工作，执行中断任务；轮询则为cpu主动，定时查询硬件设备的状态，是否处理硬件请求。
categories: posts blog
---

硬件与cpu沟通有两种方式，一种是中断，另一种是轮询。中断是硬件主动，产生中断信号，中断控制器将信号传递给cpu，cpu停下手中的工作，执行中断任务；轮询则为cpu主动，定时查询硬件设备的状态，是否处理硬件请求。
<!-- more -->

硬件中断又分为两类：

+ **Non-maskable interrupts [NMI]（不可屏蔽中断）**

不可屏蔽中断针对的是严重的硬件错误，比如内存错误，传感器错误等等。出现这种中断一般需要硬件工程师介入，检测错误类型。

+ **Maskable interrupts（可屏蔽中断）**

对于IO密集型的服务来说，网卡中断会非常的频繁，每一次硬件中断都需要CPU作出响应，大量的中断其实是一件比较消耗CPU的操作。
如果中断又都集中在某一个cpu上，那么对服务的影响就比较大，这时就需要将中断分散到各个cpu上均衡处理，由此就有了下面要介绍的网卡多队列,以及相关的优化技术。

#### 1. RSS（Receive Side Scaling）
RSS需要硬件支持，通过`lspci -vvv | grep -i msi-x`命令来查看Ethernet controller中是否有MSI-X，Enable+，TabSize > 1来判断是否硬件支持。

RSS的原理，简单描述就是**通过过滤规则将网络包分发到不同的队列，每个队列通过关联的中断由指定的不同CPU处理。**
使用RSS，既能缩短单个队列长度（降低延迟），又能均衡cpu的处理能力（不至于某个cpu达到瓶颈）。
下面一一解释上面的关键字：过滤规则，队列，中断，CPU

+ **过滤规则**

过滤规则就是hash函数，对IP packet或TCP segment的header进行hash，分发到不同的队列。

+ **队列关联中断号**

通过`cat /proc/interrupts`查看网卡队列对应的中断号。

```
//查看各cpu中断类型
[root@XX-0-79-core ~]# cat /proc/interrupts
            CPU0       CPU1       CPU2       CPU3
   0:         57          0          0          0   IO-APIC-edge      timer
   1:         10          0          0          0   IO-APIC-edge      i8042
   6:          3          0          0          0   IO-APIC-edge      floppy
   8:          0          0          0          0   IO-APIC-edge      rtc0
   9:          0          0          0          0   IO-APIC-fasteoi   acpi
  12:        144          0          0          0   IO-APIC-edge      i8042
  61:         2346          0          0          0   PCI-MSI-edge      eth0-0
  62:          0         208711        0         0   PCI-MSI-edge      eth0-1
  63:         0          0                 0     147535          PCI-MSI-edge      eth0-2
```

 第1栏表示中断号，第2-5栏表示在CPU0-3上处理的中断数量，第6栏表示中断控制器，APIC表示高级可编程中断控制器。不支持APIC的控制器，不能完成中断绑定，默认发往cpu0：

 > Without an IO-APIC, interrupts from hardware will be delivered only to the CPU which boots the operating system (usually CPU#0)

 最后一栏表示中断类型，timer时钟中断，eth0-0，eth0-1即是0号网卡的0号队列和1号队列，从表中可以看出，eth0-0，eth0-1，eth0-2分别对应中断号61，62，63。
 
+ **中断号与CPU亲和**

将网卡中断号通过CPU亲和度绑定到某个CPU上即可，具体绑定方式可以查看linux文档Documentation/IRQ-affinity.txt。每个队列都关联一个中断号，当一个包到达队列之后，网卡就触发该队列关联的中断请求，通过刚刚绑定的CPU处理中断。

中断号的大小即决定了CPU处理中断请求的优先级，中断号越小，优先级越高。

```
IRQ 0 — **system timer (cannot be changed)**
IRQ 1 — **keyboard controller (cannot be changed)**
IRQ 3 — serial port controller for serial port 2 (shared with serial port 4, if present);
IRQ 4 — serial port controller for serial port 1 (shared with serial port 3, if present);
IRQ 5 — parallel port 2 and 3 or sound card;
IRQ 6 — floppy disk controller;
IRQ 7 — parallel port 1. It is used for printers or for any parallel port if a printer is not present.
```

 如果手动设置了CPU亲和度绑定，需要将irqbalance后台进程关掉（`service irqbalance status；service irqbalance stop`），因为它会自动将不同的中断交给各个cpu处理，会覆盖掉手动设置值。

对于延迟特别敏感，或者接收中断处理已经成为瓶颈的系统应该开启RSS。但是应该控制队列在cpu物理核上的数量。

> Per-cpu load can be observed using the mpstat utility, but note that on processors with hyperthreading (HT), each hyperthread is represented as a separate CPU. For interrupt handling, HT has shown no benefit in initial tests, so limit the number of queues to the number of CPU cores in the system.

#### 2. RPS（Receive Packet Steering）	
    
RPS可以看做是RSS的软件实现。原理简单描述为对`数据流hash分类`，并将软中断负载均衡到各CPU。不会增加网卡的硬中断速率，但是引入了处理器间的中断。
	
**优化方式：**
	
调整`/sys/class/net/[device]/queues/rx-[num]/rps_cpus`的值,rps_cpus默认为0000，每一位表示16进制位。000f表示使用cpu0，cpu1，cpu2，cpu3。
	
#### 3. RFS（Receive Flow Steering）

解决处理网卡数据流的cpu与接受该数据流的应用程序所运行的cpu不一致，造成cpu切换的问题。保证程序运行的CPU和软中断处理CPU是同一个，`提升CPU缓存命中率，降低网络延迟`。

**优化方式：**

要开启RFS，需要修改两个参数，	

```
/proc/sys/net/core/rps_sock_flow_entries
/sys/class/net/[device]/queues/rx-[num]/rps_flow_cnt
```
设置rps_sock_flow_entries的值为期望获得的最大并发连接的数量（该值必须为2的幂，如果不是，系统设置为向上最接近的2的幂），rps_flow_cnt的值为rps_sock_flow_entries/N，N表示设备接收队列的数量，对于单队列两者设置为同一值。

#### 4. XPS（Transmit Packet Steering）

针对网卡发送时的优化，RFS是根据包接收的队列来选择cpu，而XPS是根据cpu来选择要发送的网卡队列。

**优化方式：**

修改`/sys/class/net/[device]/queues/tx-[num]/xps_cpus`

注意RPS和RFS在linux 内核版本2.6.35才被引入，2.6.38及其以上才包含XPS。

![hardware-interrupt](/images/hardwareinterrupt/hardware-interrupt.png)

##### 参考

[1] [http://blog.netzhou.net/?p=181](http://blog.netzhou.net/?p=181)

[2] [http://hellojava.info/?p=238](http://hellojava.info/?p=238)

[3] [http://blog.csdn.net/wyaibyn/article/details/14109325](http://blog.csdn.net/wyaibyn/article/details/14109325)

[4] [https://lwn.net/Articles/412062/](https://lwn.net/Articles/412062/)

