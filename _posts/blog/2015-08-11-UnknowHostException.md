---
layout: post
title: UnknowHostException
category: blog
---

## 1 **问题**

  团队使用Docker中的测试环境，在新起的测试环境中启动消息队列项目，项目启动失败，抛出如下异常：
{% highlight java %}
UnknowHostException：
	java.net.UnknownHostException: e7a05a2abfc5: e7a05a2abfc5: 未知的名称或服务
{% endhighlight%}

## 2 **分析**

  异常信息栈显示抛出异常的地方在：`InetAddress localHost = InetAddress.getLocalHost();`
<!-- more -->
  跟踪源代码发现`InetAddress.getLocalHost()`会调用**Inet6AddressImpl**或者**Inet4AddressImpl**的native方法：

 	public native String getLocalHostName() throws UnknownHostException;

  该native方法直接执行linux的系统调用gethostname。系统调用gethostname与在终端执行`$ hostname`或者`$ cat /etc/hostname`的结果一致。获得hostname后，比较该hostname的值，如果为localhost，则执行`impl.loopbackAddress();`获得localhost对应的ip地址。

{% highlight java %}
if (local.equals("localhost")) {
              return impl.loopbackAddress();
      } else {
	......
synchronized (cacheLock) {
              long now = System.currentTimeMillis();
              if (cachedLocalHost != null) {
                  if ((now - cacheTime) < maxCacheTime) // Less than 5s old?
                      ret = cachedLocalHost;
                  else
                      cachedLocalHost = null;
              }
              // we are calling getAddressesFromNameService directly
              // to avoid getting localHost from cache
              if (ret == null) {
                  InetAddress[] localAddrs;
                  try {
                      localAddrs =
                          InetAddress.getAddressesFromNameService(local, null);
                  } catch (UnknownHostException uhe) {
                      // Rethrow with a more informative
		UnknownHostException uhe2 =
                          new UnknownHostException(local + ": " +
                                                   uhe.getMessage());
                      uhe2.initCause(uhe);
                      throw uhe2;
                  }
                  cachedLocalHost = localAddrs[0];
                  cacheTime = now;
                  ret = localAddrs[0];
              }
          }
}
{% endhighlight %}   

  如果不是localhost，则会读取缓存;如果缓存不存在，调用`InetAddress.getAddressesFromNameService(local, null)`。该方法调用native的`lookupAllHostAddr`方法，该方法会在/etc/hosts里面查找对应host的ip地址，如果没有则抛出UnknowHostException。

## 3 **解决方法**

  知道了异常发生的原因，那么解决方法只需要在/etc/hosts里面添加一项

`127.0.0.1 e7a05a2abfc5`

即可。另外还可以选择将`hostname`的值改为`localhost`的方式来解决问题。

![thatslove](/images/unknowhostexception/thatslove.jpg)
