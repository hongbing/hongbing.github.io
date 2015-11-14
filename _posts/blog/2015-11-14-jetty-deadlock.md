---
layout: post
title: jetty6 deadlock
category: blog
nav:
   - name: java
   - link: "#java"
---

## 1 背景
业务方反应消息队列最近几天输出的消息量变少了（良好的监控系统应该在业务方感知到故障之前将问题暴露出来），进入到前端机发现，多达20多万的TCP连接状态都处于`CLOSE_WAIT`状态，且TCP socket的RECV-Q中还有100多个包没有被处理。
我们使用的环境是：

+ java
  java version "1.6.0_24"
  Java(TM) SE Runtime Environment (build 1.6.0_24-b07)<!-- more -->
  Java HotSpot(TM) 64-Bit Server VM (build 19.1-b02, mixed mode)

+ jetty
  jetty 6.1.26

## 2 分析
看到大量的tcp连接都处于CLOSE_WAIT状态，初步分析了一下网络状况，如果服务端处于CLOSE_WAIT状态，说明是客户端首先发起了关闭请求。但是在这个项目中，客户端与服务端之间采用的是jetty长连接，服务端在服务10分钟之后主动断开连接，客户端重新发起请求。明显，现在出现的情况与设计是不符合的。为了尽快恢复服务状况，释放服务端的连接，于是重启了服务端程序。重启之后，服务暂时恢复正常。在重启之前，使用jstack dump了一份当前java进程的线程栈,发现在dump的文件最后，检测到了deadlock，内容如下：

```
 Java stack information for the threads listed above:
"1854411210@qtp-1306606930-148":
	at org.mortbay.io.nio.SelectorManager$SelectSet.scheduleIdle(SelectorManager.java:795)
	- waiting to lock <0x00000006fd82b228> (a org.mortbay.io.nio.SelectorManager$SelectSet)
	at org.mortbay.io.nio.SelectChannelEndPoint.scheduleIdle(SelectChannelEndPoint.java:159)
	at org.mortbay.io.nio.SelectChannelEndPoint.blockWritable(SelectChannelEndPoint.java:293)
	- locked <0x0000000777e8d338> (a org.mortbay.jetty.nio.SelectChannelConnector$ConnectorEndPoint)
	at org.mortbay.jetty.AbstractGenerator$Output.blockForOutput(AbstractGenerator.java:544)
	at org.mortbay.jetty.AbstractGenerator$Output.flush(AbstractGenerator.java:571)
	at org.mortbay.jetty.HttpConnection$Output.flush(HttpConnection.java:1010)
	at cn.xxxx.xxxxx.channel.DataOutputChannel.process(DataOutputChannel.java:23)
	at cn.xxxx.xxxxx.channel.DataChannel.handler(DataChannel.java:40)
	at cn.xxxx.xxxxx.channel.DataChannelFactory.handler(DataChannelFactory.java:27)
	at cn.xxxx.xxxxx.processor.ClientProcessor.outputDataMessage(ClientProcessor.java:287)
	at cn.xxxx.xxxxx.processor.ClientProcessor.outputDataMessage(ClientProcessor.java:259)
	at cn.xxxx.xxxxx.processor.ClientProcessor.handler(ClientProcessor.java:82)
	at cn.xxxx.xxxxx.web.DataServlet.service(DataServlet.java:34)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:820)
	at org.mortbay.jetty.servlet.ServletHolder.handle(ServletHolder.java:511)
	at org.mortbay.jetty.servlet.ServletHandler.handle(ServletHandler.java:390)
	at org.mortbay.jetty.security.SecurityHandler.handle(SecurityHandler.java:216)
	at org.mortbay.jetty.servlet.SessionHandler.handle(SessionHandler.java:182)
	at org.mortbay.jetty.handler.ContextHandler.handle(ContextHandler.java:765)
	at org.mortbay.jetty.webapp.WebAppContext.handle(WebAppContext.java:440)
	at org.mortbay.jetty.handler.ContextHandlerCollection.handle(ContextHandlerCollection.java:230)
	at org.mortbay.jetty.handler.HandlerCollection.handle(HandlerCollection.java:114)
	at org.mortbay.jetty.handler.HandlerWrapper.handle(HandlerWrapper.java:152)
	at org.mortbay.jetty.Server.handle(Server.java:326)
	at org.mortbay.jetty.HttpConnection.handleRequest(HttpConnection.java:542)
	at org.mortbay.jetty.HttpConnection$RequestHandler.headerComplete(HttpConnection.java:926)
	at org.mortbay.jetty.HttpParser.parseNext(HttpParser.java:549)
	at org.mortbay.jetty.HttpParser.parseAvailable(HttpParser.java:212)
	at org.mortbay.jetty.HttpConnection.handle(HttpConnection.java:404)
	at org.mortbay.io.nio.SelectChannelEndPoint.run(SelectChannelEndPoint.java:410)
	at org.mortbay.thread.QueuedThreadPool$PoolThread.run(QueuedThreadPool.java:582)

"965223859@qtp-1306606930-25 - Acceptor14 SelectChannelConnector@0.0.0.0:8082":
	at org.mortbay.io.nio.SelectChannelEndPoint.updateKey(SelectChannelEndPoint.java:322)
	- waiting to lock <0x0000000777e8d338> (a org.mortbay.jetty.nio.SelectChannelConnector$ConnectorEndPoint)
	at org.mortbay.io.nio.SelectChannelEndPoint.close(SelectChannelEndPoint.java:456)
	at org.mortbay.jetty.nio.SelectChannelConnector$ConnectorEndPoint.close(SelectChannelConnector.java:362)
	at org.mortbay.io.nio.SelectChannelEndPoint.idleExpired(SelectChannelEndPoint.java:174)
	at org.mortbay.io.nio.SelectChannelEndPoint$IdleTask.expire(SelectChannelEndPoint.java:489)
	at org.mortbay.thread.Timeout.tick(Timeout.java:137)
	- locked <0x00000006fd82b228> (a org.mortbay.io.nio.SelectorManager$SelectSet)
	at org.mortbay.thread.Timeout.tick(Timeout.java:153)
	at org.mortbay.io.nio.SelectorManager$SelectSet.doSelect(SelectorManager.java:762)
	at org.mortbay.io.nio.SelectorManager.doSelect(SelectorManager.java:192)
	at org.mortbay.jetty.nio.SelectChannelConnector.accept(SelectChannelConnector.java:124)
	at org.mortbay.jetty.AbstractConnector$Acceptor.run(AbstractConnector.java:708)
	at org.mortbay.thread.QueuedThreadPool$PoolThread.run(QueuedThreadPool.java:582)
```

另一方面，跟业务方要了一份客户端的代码。发现客户端确实在应用层面主动关闭了连接（why，不按照设计来？？），这样也解释了为什么会在服务端出现CLOSE_WAIT状态。

接下来看看服务端代码的deadlock:

跟踪调用链路可以看到死锁的发生，一个线程锁在了`org.mortbay.io.nio.SelectorManager$SelectSet`上，并且等待`rg.mortbay.jetty.nio.SelectChannelConnector$ConnectorEndPoint`的解锁，另一个则相反。

对于线程`1854411210@qtp-1306606930-148`，如果没有等待`org.mortbay.io.nio.SelectorManager$SelectSet`解锁，那么会发生什么呢？看看下面的代码：

```java
   if (!_generator._endp.blockWritable(_maxIdleTime))
   {
       _generator._endp.close();
       throw new EofException("timeout");
   }
                
   _generator.flush();
```

等待解锁的调用走到了`_generator._endp.blockWritable()`方法的内部，如果没有形成死锁，那么等到`_maxIdleTime`时间到，要么执行flush操作，将数据传输到对端，要么关闭连接，并抛出**timeoue**异常。因此不会产生前文所描述的服务端的大部分请求连接都处于CLOSE_WAIT状态，导致无法提供服务。

知道了产生问题的原因，在网上搜索关键词**jetty6 deadlock**会有不少这方面的bug report。网上也一直以为是[jetty6的bug](https://bugs.eclipse.org/bugs/show_bug.cgi?id=357318)，然而经过jetty相关开发人员的验证，最终由oracle方面证实是[jdk的bug](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=7130796)，且该bug已经在jdk8-b28得到修复。

接下来我们需要做的就是升级系统到jdk8-b28及其以上，同时将jetty升级到jetty8。

![deadlock](/images/jettydeadlock/jettydeadlock.jpg)
