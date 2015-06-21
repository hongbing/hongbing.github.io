---
layout:     post
title: ETCD vs. Consul
category: project
description: etcd和consul均采用raft协议来保证server之间的数据一致性，但是consul还采用了gossip协议管理成员和传播消息。
---

### 异同
-----------------------------
#### 1. 协议
+ Etcd和consul均采用raft协议来保证server之间的数据一致性，但是consul还采用了gossip协议管理成员和传播消息。

+ Etcd中的RPC消息格式都是protobuf格式，Consul的RPC消息格式采用MsgPack格式。

+ Etcd支持HTTP操作接口，Consul支持HTTP和DNS两种操作接口

#### 2. 架构
+ Etcd不支持多数据中心，consul支持多数据中心

+ Etcd集群中的server存在三种角色：leader，candidate和follower；而consul集群中只有agent，agent可以以client和server两种形式存在。

+ Etcd可以以proxy和etcd两种方式启动，proxy只负责请求的转发，是请求转发代理的角色，不参与raft一致性协议（关于请求转发这方面功能Etcd的proxy类似于consul的client）。

+ Etcd和Consul都建议server数量在3，5，7个左右。Consul的client数量可以多达几百上千。

#### 3. 数据处理
+ `Consul集群中的所有请求都会重定向到server，对于非leader的server会将写请求转发给leader处理`。对于读请求支持3种一致性模式：
	+ **default**
	给leader一个time window，在这个时间内可能出现两个leader（split-brain情况下），旧的leader仍然支持读的请求，会出现不一致的情况。
	+ **consistent**
	完全一致，在处理读请求之前必须经过大多数的follower确认leader的合法性，因此会多一次round trip。
	+ **stale**
	允许所有的server支持读请求，不管它是否是leader。
	
+ **默认情况下**，`Etcd集群中的每个member都可以处理client的读请求`，因此增加成员数量可以增大读请求的吞吐量。而写请求必须由leader处理，`每个写请求都需要复制到大多数的follower才可以提交`，因此减少member的数量可以提高写请求的性能。

对于写请求来说，两者都必须经过leader的强一致来处理。但是对于读请求，consul的default模式在etcd被认为是consistent，consul的stale模式在etcd是被当作默认的一致性模式。
Etcd使用强一致性读需要添加参数`quorum=true`

#### 4. 健康检测
+ Consul支持健康检测，包括应用级和系统级两种健康检测。

+ Etcd不支持健康检测。

#### 5.  压测
+ Etcd支持单实例1000次/s的写请求

+ Consul支持几百到上千每秒的写请求，具体看网络带宽和硬件配置。


####6. 安全
+ Consul支持ACL，Etcd不支持（TODO）

+ Etcd client与server或者peer之间的通信支持SSL/TLS认证


#### 7. KV存储
+ Etcd和Consul均采用类似与*nux文件目录格式存储，Etcd支持隐藏的目录节点（以`/—`开始的key，不知道做什么用，在V3版可能去掉），etcd存储的value可以直接采用json文件，xml文件，txt文件等。

+ Etcd中的value都是以json格式存储（空间占用比较大，Etcd的V3.0可能支持二进制存储）。 


#### 8. 变更通知
+ Etcd使用long polling监听某一个key的变化，当变化发生时将变化key的相关信息推送给监听key的client。同时，还可以监听最近1000次以内的key的变化情况（X-Etcd-Index表示当前获取的key的index）。

+ Consul可以监听`key`, `keyprefix`, `services`, `service`, `event`, `checks`, `nodes`。当监听到变更发生后，使用外部的handler处理。handler可以是任意可执行的进程，比如python脚本等等。
Consul某些节点的监听使用long polling等待直到监听的X-Consul-Index位置发生变化。long polling也可以使用`wait`参数设置监听的时间（默认10min）

####9. 节点变更或故障处理
+ Etcd中的server故障（硬件故障等难以恢复的故障），应该尽快用新的服务器替换掉故障的机器，否则quorum机制可能产生影响。需要手动执行命令移除和添加新的member。如果故障机器中的数据大于50MB，最好将数据copy到新的机器中，否则leader会将store中的所有数据传输给该member，将会长时间占用带宽。

+ Consul遭遇网络故障或者agent crash导致一部分node不可用，这些unreachable node会自动被标记成failed。gossip 协议会在一定时间内（72小时）尝试重连failed的节点。


#### 10. 请求超时
+ Etcd不对get request做超时处理，因为get request为non-blocking way。watch request也不设置超时。只对Delete, Put, Post, QuorumGet requests设置超时时间，超时发生要么请求的server故障，要么集群中的大多数server故障。

+ Consul对blocking query有超时机制。

#### 11. 集群节点发现
+ Etcd支持三种节点发现功能：
	+ **static**
	已知集群节点个数，IP地址等信息，通过-initial-cluster参数指定集群中的节点信息
	```
-initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 -initial-cluster-state new
	```
	
	+ **discovery**
	Etcd自身的节点发现功能，使用自定义的注册目录
    ` curl -X PUT https://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83/_config/size -d value=3`
    上面的命令在已有的集群上注册node size，命令表明集群中有3个node，只有当3个node都注册后集群才开始工作，如果注册的node多于3个，那多余的都降为proxy模式启动。
    如果没有权限获取已有的集群信息，可以使用公共的discovery url：`https://discovery.etcd.io`
	+ **DNS SRV records**

+ Consul通过`join`命令将node加入到集群中，每添加一个node都会加入到gossip pool，通过gossip protocol将新加入的node信息传输到集群中的所有node。


#### 12. 统计数据
+ Etcd支持node的统计数据，包括latency, bandwidth和uptime， Store的统计数据。

+ Consul支持对libraries和subsystem的性能统计信息。

#### 13. 输出信息格式
+ Etcd和Consul均采用json格式输出