---
layout: post
title: 动态变更log4j日志级别
description: 日志级别的动态控制可以有效的解决各个项目成员对于某日志应该设置什么级别而争论的烦恼，同时对线上问题的排查也会起到非常重要的作用。
categories: posts blog
---

log4j的日志级别包括ALL，TRACE，DEBUG，INFO，WARN，ERROR, FATAL，OFF。日志级别从前往后越来越严格，ALL和OFF（日志级别的两个极端）在项目开发中都使用的比较少，可以说是基本不使用。每个级别在什么样的场景下使用这里就不多说了。下面说一下，为什么需要日志级别的动态变更？

不管是第三方开发包，还是自己开发的项目工程，在调试的过程中都会有大量的TRACE，DEBUG日志，<!-- more --> 然而上线之后这些TRACE，DEBUG日志绝大部分都不需要出现在业务日志中，一是日志量可能较大，占用存储空间，消耗性能，二是大量调试的日志会对线上业务相关的日志造成干扰。因此，线上日志的级别就被设定为INFO以及更严格的WARN。但是，当某一天线上某个功能不正常了，特别是第三方组件的问题，我们就抓瞎了。到底走到哪一步了，是什么样的结果，完全不知道，有TRACE，DEBUG日志也看不见，只得重新打包上线，更改日志输出等级。即便这样也只能粗略的把可能需要更改的包都改了，不能精确的定位到需要改哪个包，哪个类的日志输出等级。当解决问题后，还需要再上一次线把日志输出等级改回去。这里还需要考虑的是，有的问题一旦重启就复现不了了。要排查问题就不能轻易的重启。

另一方面，不同的开发人员对于日志输出等级的控制理解不同，对于日志输出也有自己的认识。有的开发在输出日志时，会尽可能多的将日志都输出，不管什么方法，都在方法入口和出口都输出一句日志。我是不建议这种做法，一是在很多情况下都是对日志的一种浪费，二是不加思考的开发，对线上服务是一种很危险的行为。对于这样的情况，需要开发考虑在需要输出日志的地方才输出。如果线上出现问题，在疯狂的刷某一些相关的日志，但是某些日志特别长，信息特别多，对解决问题又没有帮助，这时我们就需要关闭一些日志的输出。

基于上面的两种情况，都需要能够在线上动态的调整日志输出等级。

再多说一点关于如何进行业务日志输出的问题。我的理解是，凡是写方法都应该有入口日志，入口日志应该详细描述入口参数以及可能影响写方法结果的变量，同时可以在方法出口增加返回值相关的日志；读方法可以选择性的加日志，对于按页读取数据的方法，返回值可能会很多，一般不要输出大面积的日志，或者把日志输出级别调低。

Log4J2的文档里面有很详细的设置，更新logger输出等级的代码。我这里贴出项目里面使用的：
{% highlight java %}

 private static final ConcurrentHashMap<String, Object> loggerContainer = new ConcurrentHashMap<>();
 private static final LoggerContext loggerContext = (LoggerContext) LogManager.getContext(false);

 public static Pair<Boolean, String> setLoggerLevel(String loggerName, String level) {
        logger.warn("set logger level, loggerName:{}, level:{}", loggerName, level);

        LoggerConfig loggerConfig = (LoggerConfig) loggerContainer.get(loggerName);
        Level targetLevel = StringUtils.isBlank(level) ? Level.INFO : Level.toLevel(level);

        if (loggerConfig == null) {
            logger.warn("no loggerConfig for loggerName:{}", loggerName);
            return Pair.makePair(false, loggerName + " logger not exist.");
        }
        loggerConfig.setLevel(targetLevel);
        loggerContext.updateLoggers();
        return Pair.makePair(true, "set logger: " + loggerName + ", level: " + targetLevel.name());
    }

{% endhighlight%}

loggerName可以是某个包名，比如com.appache.zookeeper，设置之后会将该包下的所有日志等级都变更。当你只需要变更某个类中的输出日志等级时，指定loggerName为某个具体的类名即可。

在线上使用log4j2的xml配置时，需要配置不同包的输出日志等级，以及输出到某个文件。配置log4j2的root Appender的level为INFO，然后设置输出文件的Appender的ThresholdFilter的level为DEBUG。也即 该文件可以输出TRACE及其以下的任何日志，但是默认的Root Appender的级别为INFO，所以只有info及其以下等级日志才会输出到文件。这样当更改线上某个类，或者某个包的级别时，都会输出到文件。

注意：动态的变更了日志输出等级，当服务重启后动态变更的会失效，仍旧保持xml里配置的情况。所以特别适合临时调整线上日志级别。

##### 参考
[1] [https://logging.apache.org/log4j/2.0/manual/architecture.html](https://logging.apache.org/log4j/2.0/manual/architecture.html)

[2] [https://tech.meituan.com/change_log_level.html](https://tech.meituan.com/change_log_level.html)


