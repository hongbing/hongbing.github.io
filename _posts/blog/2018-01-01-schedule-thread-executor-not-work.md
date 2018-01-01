---
layout: post
title: 定时任务怎么不定时执行了
description: 定时器任务在执行一段时间之后停止了，为什么呢？
categories: posts blog
---

2017年的元旦假期，测试反馈说微博里红豆直播间的观看人数在一段时间内都没有变化。接到问题的第一反应是不是前一天上线导致的？一个好好的系统功能，突然在某一天不正常工作了，如果恰巧前面有一次有上线变更，那么90%的原因就是上线变更导致的。我赶紧看一下前一天的上线是否涉及到这一块儿的内容？<!-- more --> 经过查看比对，发现上线的功能完全不涉及这里，于是否定了前面的假设。

微博里红豆直播间展示的计数是由我们定时推送过去的，现在就需要去看看我们的推送逻辑是否有问题了。计数的推送逻辑很简单，如果直播间的观看人数有变化，就发送一条kafka消息，消费端接收到kafka消息然后定时同步到微博。定时逻辑使用的是ScheduledThreadPoolExecutor线程池，每隔30秒同步一次数据。该ScheduledThreadPoolExecutor的初始化和定时启动在kafka消费线程的初始化逻辑里面。即只要kafka消费端线程启动了，ScheduledThreadPoolExecutor就会启动定时任务。

```
        scheduledExecutor = new ScheduledThreadPoolExecutor(50, new TaskThreadFactory("room_counters_change"));
        StatLog.registerExecutor("room_counters_change", scheduledExecutor);

        scheduledExecutor.scheduleWithFixedDelay(() -> {
            logger.info("start execute scheduled sync room. size:{}", cachedRoomIds.size());
            List<Long> roomIds = swapRoomIds();
            Map<Long, RoomInfo> batchRoomInfo = roomService.getBatchRoomInfoWithStatistic(roomIds);
            if (batchRoomInfo != null && batchRoomInfo.size() > 0) {
		// 向微博同步直播间的计数
                syncRoomCounterToWeibo.sync(batchRoomInfo);
            }

        }, 30, 30, TimeUnit.SECONDS);
```

接下来的排查逻辑就只有到线上服务器查看日志了。随便登录了一台服务器199，搜索日志“start execute scheduled sync room. size:0”，发现size的值都是0，说明确实没有同步直播间的数据过去。现在需要排查的就是为什么没有直播间的数据？

直播间的数据来自于kafka消费端接收到的数据，代码如下：

```
    @Override
    @KafkaAnnotation(topic = KafkaTopic.SYNC_ROOM_COUNTERS_CHANGE)
    protected void process(MQEntry entry) {
        Collection<Long> roomIds = (Collection<Long>) entry.getBody();
        this.cachedRoomIds.addAll(roomIds);
        logger.debug("kafka consumer handle sync_room_counters_change, entry:{}, cachedRoomIds:{}", JsonConverter.format(entry), JsonConverter.format(cachedRoomIds));
    }
```

在线上打开了debug日志之后，发现并没有“kafka consumer handle sync_room_counters_change”日志出现，但是另外可以确定的是上层一定发送了kafka消息。那么是不是kafka消费端出了问题？对于kafka的消费线程，我们有开关可以启停该线程是否消费kafka消息，是不是上一次上线的时候启动消费的开关打开失败了？于是，手动执行命令打开199上的消费kafka消息的开关，但是仍然没有“kafka consumer handle sync_room_counters_change”日志出现，查看日志开关确实已经打开了，那么为什么没有消费到消息呢？问题是不是出在kafka本身？

于是打开了kafka manager监控页面，看到“sync_room_counters_change”的topic消费端只有两个partition，两个partition分别由211和212机器消费，且没有堆积。那么说明kafka消费线程没有问题。由于线上机器有16台，于是将“sync_room_counters_change”的partition增加到16个（默认情况下新的topic的partition是2个，可以修改kafka的配置文件改变默认值）。做了这个变更后，所有的机器都分得一个partition，开始消费消息，同时在除211和212之外的其余机器上搜索日志看到“start execute scheduled sync room. size:20”，size后面的值不在是0了，说明已经可以同步数据了，服务正常了。

那么，partition不够是不是问题的根本原因呢？是不是问题得到了根本的解决？通过查看日志还有一个异常的情况：211和212两台机器上，始终没有“start execute scheduled sync room. size”日志出现，即便在增加partition到16个之前也没有日志出现，这又是怎么回事儿？服务恢复正常的根本原因是什么？

我们重新来梳理一下现场：

+ 一开始只有211和212在消费“sync_room_counters_change”的topic消息，没有堆积，消费正常，但是没有定时任务的入口日志。
+ 其余机器有定时任务的入口日志，但是size的大小都为0。
+ 当将partition增加到16个之后，其余机器定时器的入口日志size的大小都不为0了，推送数据正常
+ 当将partition增加到16个之后，211和212机器仍然没有定时任务的入口日志。

从梳理的现场来看，增加partition能够让服务恢复正常（也许是临时），但是并不是服务不正常的真正原因，因为211和212并没有出现什么变化。接下来着重分析211和212机器。

由于没有定时器的入口日志，猜测是不是定时器没有启动？

```
StatLog.registerExecutor("room_counters_change", scheduledExecutor);
```

上面这一行代码可以记录Executor的任务执行情况，通过日志看到211上的executor总共提交25个任务，完成了25个任务，然后就再没有新的任务执行了，而212上的executor总共提交了46个任务，完成了46个任务，接着也没有新的任务了。那么说明定时器是执行了一段时间之后才出问题的。也就否定了前面定时器没有启动的假设。另外查询前一天上线之后的日志，发现有“start execute scheduled sync room. size”日志出现，而且size的值不为0，这也佐证了定时器已经启动。

综合上面的分析，到这里基本可以确定出现问题的原因了:

> 红豆直播间的计数通过定时器推送给微博，kafka的消费线程里面负责启动定时器executor，消费端只有两台机器211和212，这两台机器中的定时器一开始功能正常，每30秒触发一次，当211执行到第25次，212执行到46次的时候，出现了某种情况导致定时器不再定时执行了，没有继续向微博推送计数，因此计数功能不正常。
当partition增加到16个时，其他机器得到了消费的机会，又开始向微博推送计数，服务恢复正常。

接下来就是找出scheduledExecutor.scheduleWithFixedDelay不定时执行的原因了。
经过一番搜索，在参考文章[java-executor-service-tutorial](http://www.baeldung.com/java-executor-service-tutorial)中发现这样一段话：
> According to the scheduleAtFixedRate() and scheduleWithFixedDelay() method contracts, period execution of the task will end at the termination of the ExecutorService or if an exception is thrown during task execution.
根据scheduleAtFixedRate()和scheduleWithFixedDelay()方法的约定，任务的执行周期将在ExecutorService终止时或者任务执行时有异常抛出的时候结束。

通过这篇文章的指引与同事的沟通，找到了scheduledExecutor.scheduleWithFixedDelay执行过程中roomService.getBatchRoomInfoWithStatistic(roomIds)方法因为超时抛出异常的日志，确定了是因为异常情况导致了定时器不再定时执行，同时用test case也模拟了定时器遇到异常不定时执行的场景。
修复问题的方式反而显得简单了，在scheduledExecutor.scheduleWithFixedDelay的执行方法里面加上try...catch。

同时得出一个结论：`在scheduledExecutor的执行方法里面都应该用try...catch，避免因为异常导致定时器终止。`

在这次排查问题的过程中，有一些思考：

恢复故障仍然是第一位的，遇到问题需要依靠经验及时把不相关的因素排除，不要纠结于一个点上，可以不断提出假设，并用多个方面的证据（日志，现象等）佐证你的判断。
线上执行了某个操作，是否有相应的结果出来？服务恢复了，问题的根本原因是不是找到了？本次排查过程中，增加partition的操作确实让服务恢复了，但是并不是问题出现的根本原因。
要深入细节，追根溯源，到底哪个是原因，哪个是结果。


##### 参考
[1] [http://www.baeldung.com/java-executor-service-tutorial](http://www.baeldung.com/java-executor-service-tutorial)

