---
layout: post
title: 20171015一周坑
description: 坑
categories: posts blog
---

##### 1. disconf加载类的坑
在做push工程的时候，使用disconf动态配置极光的masterSecret和key，但是一直不成功，配置无法更新，始终使用的是默认的配置值。disconf下载的本地文件里面，各项配置都是动态更新的，而且在其他工程里面同样的使用方式都没有问题，判断是因为配置类没有被disconf加载到导致的。
为什么没有被加载到？ <!-- more -->

disconf相关配置如下：

```
 <bean id="disconfMgrBean" class="com.baidu.disconf.client.DisconfMgrBean"
          destroy-method="destroy">
        <property name="scanPackage" value="com.uxin.zb,  com.uxin.commons"/>
    </bean>
    <bean id="disconfMgrBean2" class="com.baidu.disconf.client.DisconfMgrBeanSecond"
          init-method="init" destroy-method="destroy">
    </bean>
```

打开disconf的debug日志，可以看到在disconf静态扫描时候的日志
> start to scan package: com.uxin.zb,  com.uxin.commons

说明disconf已经开始要去加载这两个包，但是为什么包里面的配置相关的类没有加载进来呢？看代码也没发现什么问题，只得下载disconf的源码，在源码里面加了些DEBUG日志，然后打包在测试环境使用，最终发现问题所在。

在初始化DisconfMgrBean的时候，会根据scanPackage参数来解析需要扫描的包名，解析方法是下面的`StringUtil.parseStringToStringList()`， 参数token为逗号，source即为'com.uxin.zb,  com.uxin.commons'，因为在逗号之间多加了个空格，所以解析出来的包包括com.uxin.zb和空格com.uxin.commons, 因此在根据包名查找classpath下面相应的文件时就没有找到对应com.uxin.commons包的zb-commons.jar包，导致了配置类没有加载进来，无法使用disconf的动态配置。

一个空格导致的问题，也是disconf的一个bug。

```
public static List<String> parseStringToStringList(String source,
                                                       String token) {

        if (StringUtils.isBlank(source) || StringUtils.isEmpty(token)) {
            return null;
        }

        List<String> result = new ArrayList<String>();

        String[] units = source.split(token);
        for (String unit : units) {
            result.add(unit);
        }
        return result;
    }

```


##### 2. DRDS使用mysql别名的坑
项目中使用mybatis操作数据库，自定义了查询主播粉丝uid的最大值和最小值的mapper，最大值和最小值都使用了别名，返回结果map。在返回结果里读取粉丝uid的最大值和最小值`map.get("maxUid"), map.get("minUid")`。在测试环境里面测试通过了，上线之后就发现了NPE异常。

```
<select id="selectMaxMinByToUid" parameterType="java.lang.Long" resultType="java.util.Map">
    select max(from_uid) as "maxUid",
    min(from_uid) as "minUid"
    from user_follow
    where to_uid = #{toUid,jdbcType=BIGINT}
  </select>
```

分析测试环境与线上的差异，测试环境使用的是本机mysql数据库，线上用户关系库使用的是阿里云的drds。在线上执行mybatis解析后的sql语句
```
 mysql> select max(from_uid) as "maxUid", min(from_uid) as "minUid" from user_follow where to_uid=18xxxxxx;
+---------------+---------------+
| 'maxUid'      | 'minUid'      |
+---------------+---------------+
| 20xxxxxxxxxx1 | 17xxxxxxxxxx3 |
+---------------+---------------+
```

发现返回的map结果中的key并不是maxUid和minUid，而是'maxUid'和'minUid'，所以后续再读取这两个key的时候会抛出NPE异常。再测试一下将别名的双引号去掉，返回的key就没有单引号。同样的语句，不管是别名有没有双引号在mysql中执行时，返回的key都是不带单引号的。因此，断定是阿里云的drds给带有双引号的别名的返回结果加上了单引号。

```
mysql> select max(from_uid) as maxUid, min(from_uid) as minUid from user_follow where to_uid=188xxxxx;
+---------------+---------------+
| maxUid        | minUid        |
+---------------+---------------+
| 191xxxx111111 | 17xxxxxxx9623 |
+---------------+---------------+
```
解决方法就是将别名的双引号去掉就ok了。










