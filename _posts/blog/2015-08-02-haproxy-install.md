---
layout:     post
title: haproxy部署
category: blog
description: 记录haproxy的部署过程。
---

####1. 安装

下载haproxy源码，目前国内无法访问haproxy官网（www.haproxy.org），**Fuck GFW**。
 可以通过直接访问http://www.haproxy.org/download/1.5/src/地址，选择相应的版本下载。或者使用
  `# wget http://www.haproxy.org/download/1.5/src/haproxy-1.5.12.tar.gz`
下载，下载完成后，将其解压。
  	

		 # cd haproxy-X.Y.Z
		 # uname -a
		 # make TARGET=linux26 PREFIX=/usr/local/haproxy install


linux26,26表示linux系统的主次版本号。
执行上面的命令，输出done就正确安装完成。

####2. 配置文件

+ 修改配置

配置文件可以参考源码里面examples下面的haproxy.cfg文件，haproxy里的配置参数可到[haproxy-dconv](http://cbonte.github.io/haproxy-dconv/index.html)查看。

	  	 #cd /usr/local/haproxy
		 #mkdir conf logs
	 	 #vim /conf/haproxy.cfg

下面是参考配置

>	 global  //全局配置参数，进程级
	     	log 127.0.0.1 local0//日志输出配置，所有日志都记录在本机，通过local0输出
	     	maxconn  5120  //每个进程的最大连接数 
	     	chroot   /usr/local/haproxy  //改变当前工作目录
	        uid      100  //所属用户的uid
	        gid      100  //所属用户的gid
	        daemon  //以后台形式运行haproxy   
	        nbproc   2  //创建2个进程进去deamon模式
	        pidfile  /usr/local/haproxy/logs/haproxy.pid  //pid文件
	 defaults  
	        log     127.0.0.1  local3  err
	     	mode    http  //http七层,tcp4层,health只返回OK
	     	option  httplog  //采用httpd日志格式
	     	option  dontlognull  
	     	option  redispatch  
	     	retries 3  //三次失败认为服务不可用
	     	maxconn 3000  //默认的最大连接数
	     	contimeout 5000  //连接超时时间
	     	clitimeout 50000  //客户端超时时间
	     	srvtimeout 50000  //服务器超时时间
	     	stats enable  
	     	stats uri /admin  //统计页面的url
	     	stats auth admin:admin  //统计页面的用户名密码
	     	stats realm Haproxy \ statistic  //统计页面提示信息
	     	stats hide-version//隐藏统计页面版本信息
	 listen web_proxy *:80
		 server web1 192.168.1.128:80 check inter 5000 fall 1 rise 2  
		 server web1 192.168.1.128:8080 check inter 5000 fall 1 rise 2
	//check inter 5000 是检测心跳频率，rise 2 是2次正确认为服务可用，fall 1 是1次失败认为服务不可用，weight代表权重

global是进程级别的参数；代理参数可以设置defaults，listen，frontend和backend。listen是frontend和backend的组合，可以设置frontend的端口，backend的server节点，以及一些其他参数。

+ 配置日志环境

`# vim /etc/syslog.conf `

添加

>	local0.*        /usr/local/logs/haproxy.log 
	local3.*        /usr/local/logs/haproxy_err.log 


使用`# vim /etc/sysconfig/syslog`设置系统接收外来日志，修改
 
>	SYSLOGD_OPTIONS="-r -m 0"

+ 启动命令

>	# ./sbin/haproxy -f ./conf/haproxy.cfg

+ 重启命令

>	# ./sbin/haproxy -f ./conf/haproxy.cfg -st [haproxy.pid]

+ 停止命令

>	# killall haproxy

执行

>	 # ./sbin/haproxy -f ./conf/haproxy.cfg -sf [haproxy.pid] 

可以在修改haproxy.cfg后，快速重载变更配置。
