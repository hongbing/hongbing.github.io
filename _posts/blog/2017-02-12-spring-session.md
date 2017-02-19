---
layout: post
title: spring session源码解析
description: 存在感对于每个人的生活有多么的重要，可能平时并不是太关，其实他就是生活的全部
categories: posts blog
---

最近要在项目中做用户踢线的功能，由于项目使用spring session来管理用户session，因此特地翻了翻spring session的源码，看看spring session是如何管理的。我们使用redis来存储session，因此本文只对session在redis中的存储结构以及管理做解析。

## 1 spring session使用

Spring Session对HTTP的支持是通过标准的servlet filter来实现的，这个filter必须要配置为拦截所有的web应用请求，并且它应该是filter链中的第一个filter。Spring Session filter会确保随后调用javax.servlet.http.HttpServletRequest的getSession()方法时，都会返回Spring Session的HttpSession实例，而不是应用服务器默认的HttpSession。

<!-- more -->

spring session通过注解`@EnableRedisHttpSession`或者xml配置

```
<bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration"/>
```

来创建名为springSessionRepositoryFilter的`SessionRepositoryFilter`类。该类实现了Sevlet Filter接口，当请求穿越sevlet filter链时应该首先经过springSessionRepositoryFilter，这样在后面获取session的时候，得到的将是spring session。为了springSessonRepositoryFilter作为filter链中的第一个，spring session提供了`AbstractHttpSessionApplicationInitializer`类， 它实现了`WebApplicationInitializer`类，在onStartup方法中将springSessionRepositoryFilter加入到其他fitler链前面。

```
public abstract class AbstractHttpSessionApplicationInitializer
		implements WebApplicationInitializer {

	/**
	 * The default name for Spring Session's repository filter.
	 */
	public static final String DEFAULT_FILTER_NAME = "springSessionRepositoryFilter";
  	public void onStartup(ServletContext servletContext) throws ServletException {

			......

		insertSessionRepositoryFilter(servletContext);
		afterSessionRepositoryFilter(servletContext);
	}

	/**
	 * Registers the springSessionRepositoryFilter.
	 * @param servletContext the {@link ServletContext}
	 */
	private void insertSessionRepositoryFilter(ServletContext servletContext) {
		String filterName = DEFAULT_FILTER_NAME;
		DelegatingFilterProxy springSessionRepositoryFilter = new DelegatingFilterProxy(
				filterName);
		String contextAttribute = getWebApplicationContextAttribute();
		if (contextAttribute != null) {
			springSessionRepositoryFilter.setContextAttribute(contextAttribute);
		}
		registerFilter(servletContext, true, filterName, springSessionRepositoryFilter);
	}
}
```

或者也可以在web.xml里面将springSessionRepositoryFilter加入到filter配置的第一个。

## 2 创建spring session

RedisSession在创建时设置3个变量creationTime，maxInactiveInterval，lastAccessedTime。maxInactiveInterval默认值为1800，表示间隔1800s之内该session没有被再次使用，则删除session。每次访问都更新lastAccessedTime的值。

```
/**
* Creates a new instance ensuring to mark all of the new attributes to be
* persisted in the next save operation.
*/
RedisSession() {
	this(new MapSession());
	this.delta.put(CREATION_TIME_ATTR, getCreationTime());
	this.delta.put(MAX_INACTIVE_ATTR, getMaxInactiveIntervalInSeconds());
	this.delta.put(LAST_ACCESSED_ATTR, getLastAccessedTime());
	this.isNew = true;
	this.flushImmediateIfNecessary();
}

public MapSession() {
	this(UUID.randomUUID().toString());
}
```

## 3 获取session

spring session在redis里面保存的数据包括：

+ SET类型的`spring:session:expireations:[min]`

  min表示从1970年1月1日0点0分经过的分钟数，SET集合的member为expires:[sessionId],表示members会在在min分钟过期。

+ String类型的`spring:session:sessions:expires:[sessionId]`

  该数据的TTL表示sessionId过期的剩余时间，即maxInactiveInterval。

+ Hash类型的`spring:session:sessions:[sessionId]`

  session保存的数据，记录了creationTime，maxInactiveInterval，lastAccessedTime，attribute。前两个数据是用于session过期管理的辅助数据结构。

应用通过getSession(boolean create)方法来获取session数据，参数create表示session不存在时是否创建新的session。getSession方法首先从请求的“.CURRENT_SESSION”属性来获取currentSession，没有currentSession，则从request取出sessionId，然后读取spring:session:sessions:[sessionId]的值，同时根据lastAccessedTime和MaxInactiveIntervalInSeconds来判断这个session是否过期。如果request中没有sessionId，说明该用户是第一次访问，会根据不同的实现，如RedisSession，MongoExpiringSession，GemFireSession等来创建一个新的session。

另， 从request取sessionId依赖具体的HttpSessionStrategy的实现，spring session给了两个默认的实现CookieHttpSessionStrategy和HeaderHttpSessionStrategy，即从cookie和header中取出sessionId。

```
@Override
public HttpSessionWrapper getSession(boolean create) {
	HttpSessionWrapper currentSession = getCurrentSession();
	if (currentSession != null) {
		return currentSession;
	}
	// 从request请求中得到sessionId
	String requestedSessionId = getRequestedSessionId();
	if (requestedSessionId != null
			&& getAttribute(INVALID_SESSION_ID_ATTR) == null) {
		S session = getSession(requestedSessionId);
		if (session != null) {
			this.requestedSessionIdValid = true;
			currentSession = new HttpSessionWrapper(session, getServletContext());
			currentSession.setNew(false);
			setCurrentSession(currentSession);
			return currentSession;
		}
		else {
			// This is an invalid session id. No need to ask again if
			// request.getSession is invoked for the duration of this request
			setAttribute(INVALID_SESSION_ID_ATTR, "true");
		}
	}
	if (!create) {
		return null;
	}

	S session = SessionRepositoryFilter.this.sessionRepository.createSession();
	session.setLastAccessedTime(System.currentTimeMillis());
	currentSession = new HttpSessionWrapper(session, getServletContext());
	setCurrentSession(currentSession);
	return currentSession;
}
```

## 4 session有效期与删除

spring session的有效期指的是访问有效期，每一次访问都会更新lastAccessedTime的值，过期时间为lastAccessedTime + maxInactiveInterval，也即在有效期内每访问一次，有效期就向后延长maxInactiveInterval。

对于过期数据，一般有三种删除策略：

1）**定时删除**，即在设置键的过期时间的同时，创建一个定时器， 当键的过期时间到来时，立即删除。

2）**惰性删除**，即在访问键的时候，判断键是否过期，过期则删除，否则返回该键值。

3）**定期删除**，即每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。

redis删除过期数据采用的是`懒性删除+定期删除`组合策略，也就是数据过期了并不会及时被删除。为了实现session过期的及时性，spring session采用了定时删除的策略，但它并不是如上描述在设置键的同时设置定时器，而是采用固定频率（1分钟）轮询删除过期值，这里的删除是惰性删除。

轮询操作并没有去扫描所有的spring:session:sessions:[sessionId]的过期时间，而是在当前分钟数检查前一分钟应该过期的数据，即spring:session:expirations:[min]的members，然后delete掉spring:session:expirations:[min]，惰性删除spring:session:sessions:expires:[sessionId]。

还有一点是，查看三个数据结构的TTL时间，spring:session:sessions:[sessionId]和spring:session:expirations:[min]比真正的有效期大5分钟，目的是确保当expire key数据过期后，还能获取到session保存的原始数据。

```
@Scheduled(cron = "${spring.session.cleanup.cron.expression:0 * * * * *}")
public void cleanupExpiredSessions() {
	this.expirationPolicy.cleanExpiredSessions();
}

public void cleanExpiredSessions() {
	long now = System.currentTimeMillis();
	long prevMin = roundDownMinute(now);

	// preMin时间到，将spring:session:expirations:[min], set集合中members包括了这一分钟之内需要过期的所有
	// expire key删掉, member元素为expires:[sessionId]
	String expirationKey = getExpirationKey(prevMin);
	Set<Object> sessionsToExpire = this.redis.boundSetOps(expirationKey).members();
	this.redis.delete(expirationKey);
	for (Object session : sessionsToExpire) {
		// sessionKey为spring:session:sessions:expires:[sessionId]
		String sessionKey = getSessionKey((String) session);
		//利用redis的惰性删除策略
		touch(sessionKey);
	}
}
```

每一次请求，spring session都会通过onExpirationUpdated()方法来更新session的过期时间， 具体的操作看下面源码的注释。

```
public void onExpirationUpdated(Long originalExpirationTimeInMilli,
			ExpiringSession session) {
	String keyToExpire = "expires:" + session.getId();
	long toExpire = roundUpToNextMinute(expiresInMillis(session));

	if (originalExpirationTimeInMilli != null) {
		long originalRoundedUp = roundUpToNextMinute(originalExpirationTimeInMilli);
		// 更新expirations:[min],两个分钟数之内都有这个session，将前一个set中的成员删除
		if (toExpire != originalRoundedUp) {
			String expireKey = getExpirationKey(originalRoundedUp);
			this.redis.boundSetOps(expireKey).remove(keyToExpire);
		}
	}

	long sessionExpireInSeconds = session.getMaxInactiveIntervalInSeconds();
	String sessionKey = getSessionKey(keyToExpire);

	if (sessionExpireInSeconds < 0) {
		this.redis.boundValueOps(sessionKey).append("");
		this.redis.boundValueOps(sessionKey).persist();
		this.redis.boundHashOps(getSessionKey(session.getId())).persist();
		return;
	}

	String expireKey = getExpirationKey(toExpire);
	BoundSetOperations<Object, Object> expireOperations = this.redis
			.boundSetOps(expireKey);
	expireOperations.add(keyToExpire);

	long fiveMinutesAfterExpires = sessionExpireInSeconds
			+ TimeUnit.MINUTES.toSeconds(5);

	// expirations:[min] key的过期时间加5分钟
	expireOperations.expire(fiveMinutesAfterExpires, TimeUnit.SECONDS);
	if (sessionExpireInSeconds == 0) {
		this.redis.delete(sessionKey);
	}
	else {
		// expires:[sessionId] 值为“”，过期时间为MaxInactiveIntervalInSeconds
		this.redis.boundValueOps(sessionKey).append("");
		this.redis.boundValueOps(sessionKey).expire(sessionExpireInSeconds,
				TimeUnit.SECONDS);
	}
	// sessions:[sessionId]的过期时间 加5分钟
	this.redis.boundHashOps(getSessionKey(session.getId()))
			.expire(fiveMinutesAfterExpires, TimeUnit.SECONDS);
}
```

## 5 参考

[1] [spring-session官网](http://docs.spring.io/spring-session/docs/current/reference/html5/)

[2] [通过Spring Session实现新一代的Session管理](http://www.infoq.com/cn/articles/Next-Generation-Session-Management-with-Spring-Session)

[3] [SpringSession原理解析](https://segmentfault.com/a/1190000004369392)

[4] [spring-session github](https://github.com/spring-projects/spring-session)
