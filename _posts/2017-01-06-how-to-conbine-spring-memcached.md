---
layout: post
title:  "spring集成memcached"
categories: memcached
tags:  spring memcached
author: hanjianchun
---

* content
{:toc}

将memcached集成到spring项目中，并封装成通用方法给项目提供缓存支持！



## 安装并启动 *memcached*

下载memcached安装包，解压安装，这里就不提了；

启动memcached服务端:

```shell
> /usr/local/bin/memcached -d -m 500 -u root -l 192.168.201.140 -p 11211 -c 256 -P /tmp/memcached.pid

-d选项是启动一个守护进程，
-m是分配给Memcache使用的内存数量，单位是MB，我这里是500MB，
-u是运行Memcache的用户，我这里是root，
-l是监听的服务器IP地址，如果有多个地址的话，我这里指定了服务器的IP地址192.168.201.140，
-p是设置Memcache监听的端口，我这里设置了11211，最好是1024以上的端口，
-c选项是最大运行的并发连接数，默认是1024，我这里设置了256，按照你服务器的负载量来设定，
-P是设置保存Memcache的pid文件，我这里是保存在 /tmp/memcached.pid，
```

## 集成到spring项目中

>三种客户端选择

memcached服务端跑在服务端上，java想访问它需要一个客户端软件，目前有三种客户端可以提供选择：spymemcached,xmemcached,java-memcached。对于这三种客户端选择哪个，开始的时候我也很困惑，所以我每一个都在项目里配置一遍看看他们的运行情况如何，最后得出了结论，spymemcached和xmemcached在我程序中遭受大量写入的时候服务器就会拒绝连接，而java-memcached依然坚挺，所以毅然投入它的怀抱里！

>首先下载jar包

由于java-memcached还没有添加到官方的maven仓库里，所以就把它放到lib下面，再引入

```xml
        <!-- memcached start -->
        <dependency> 
			<groupId>com.danga</groupId>
			<artifactId>java-memcached</artifactId>
			<version>2.6.6</version>
			<scope>system</scope>
            <systemPath>${project.basedir}/src/main/webapp/WEB-INF/lib/java-memcached-2.6.6.jar</systemPath>
		</dependency>
		<dependency>  
			<groupId>commons-pool</groupId>  
			<artifactId>commons-pool</artifactId>  
			<version>1.5.6</version>  
		</dependency>  
        <!-- memcached end -->
```

>编写memcached配置文件 *spring-context-memcached.xml*

配置文件编写完毕后要在项目启动时启动起来，在web.xml中配置spring启动时加载文件

	<!-- 加载Spring配置文件 -->
  	<context-param>
    	<param-name>contextConfigLocation</param-name>
    	<param-value>
			classpath*:/spring-context*.xml
		</param-value>
  	</context-param>

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
	xmlns:jee="http://www.springframework.org/schema/jee"
	xmlns:tx="http://www.springframework.org/schema/tx" 
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:jdbc="http://www.springframework.org/schema/jdbc" 
	xmlns:aop="http://www.springframework.org/schema/aop"
	xmlns:cache="http://www.springframework.org/schema/cache"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
	http://www.springframework.org/schema/beans/spring-beans-3.1.xsd 
	http://www.springframework.org/schema/tx 
	http://www.springframework.org/schema/tx/spring-tx-3.1.xsd 
	http://www.springframework.org/schema/jee 
	http://www.springframework.org/schema/jee/spring-jee-3.1.xsd
	http://www.springframework.org/schema/aop 
	http://www.springframework.org/schema/aop/spring-aop-3.1.xsd 
	http://www.springframework.org/schema/context 
	http://www.springframework.org/schema/context/spring-context-3.1.xsd 
	http://www.springframework.org/schema/jdbc 
	http://www.springframework.org/schema/jdbc/spring-jdbc-3.1.xsd 
	http://www.springframework.org/schema/cache 
	http://www.springframework.org/schema/cache/spring-cache-3.1.xsd"
	default-lazy-init="false">
	
	<description>Memcached Configuration</description>
	
	<!-- memcached缓存服务 -->
	<bean id="memcachedPool" class="com.danga.MemCached.SockIOPool" factory-method="getInstance"  
		init-method="initialize" destroy-method="shutDown"> 
		<constructor-arg>
			<value>memCachedPool</value>
		</constructor-arg> 
		<property name="servers">
			<list>  
				<value>192.168.201.140:11211</value>
			</list>  
		</property>
		<property name="initConn"><!-- 初始化连接数 -->
			<value>20</value>
		</property>  
		<property name="minConn"><!-- 最小连接数 -->
			<value>10</value>
		</property>  
		<property name="maxConn"><!-- 最大连接数 -->
			<value>50</value>
		</property>  
		<property name="maintSleep"><!-- 平衡线程休眠时间 -->
			<value>3000</value>
		</property>  
		<property name="nagle"><!-- Socket的参数，如果是true在写数据时不缓冲，立即发送出去 -->
			<value>false</value>
		</property>  
		<property name="socketTO"><!-- 响应超时时间 -->
			<value>3000</value>
		</property>  
	</bean>  
      
	<bean id="memCachedClient" class="com.danga.MemCached.MemCachedClient">  
			<constructor-arg><value>memCachedPool</value></constructor-arg>
	</bean>  
     
</beans>
```

> 编写memcached工具类，对外提供方法调用

```java
public class MemcachedUtil {
	static final Logger log = LoggerFactory.getLogger(MemcachedUtil.class);
	private static MemCachedClient memcachedClient = Memcached.getMemcachedClient();
	//设置放入的key前缀名
	private final static String PRO_NAME = "HJC_";
	private MemcachedUtil(){
		
	}
	/**
	 * 根据key获取缓存
	 * @param key
	 * @param defaultValue
	 * @return
	 */
	public static Object getCache(String key, Object defaultValue) {
		Object obj = memcachedClient.get(PRO_NAME+key);
		return obj==null?defaultValue:obj;
	}
	/**
	 * 放入缓存，没有过期时间
	 * @param key
	 * @param value
	 */
	public static void putCache(String key, Object value) {
		log.info("放入无永久缓存 key:【"+PRO_NAME+key+"】value:【"+value+"】");
		memcachedClient.set(PRO_NAME+key, value);
	}
	
	/**
	 * 放入缓存key-value 过期时间expire分钟
	 * @param key 关键字
	 * @param value 值
	 * @param expireSeconds 分钟
	 */
	public static void putCache(String key, Object value,int expire) {
		log.info("放入缓存过期时间：【"+expire*60+"s】 key:【"+PRO_NAME+key+"】value:【"+value+"】");
		memcachedClient.set(PRO_NAME+key, value, new Date(expire*1000*60));
	}
	
	/**
	 * 根据key删除缓存
	 * @param key
	 */
	public static void removeCache(String key) {
		log.info("删除缓存key:【"+PRO_NAME+key+"】");
		memcachedClient.delete(PRO_NAME+key);
	}
	
	/**
	 * 清空所有缓存
	 */
	public static void clearCache(){
		log.info("清空缓存");
		memcachedClient.flushAll();
	}
	
}


public class Memcached {
	
	private static MemCachedClient memcachedClient = null;
	
	public static MemCachedClient getMemcachedClient() {
		try{
			synchronized (Memcached.class) {
				if(null == memcachedClient)
					memcachedClient = SpringContextHolder.getBean("memCachedClient");
			}
		}catch (Exception e) {
		}
		return memcachedClient;
	}
}
```

>程序里调用

	工具类封装完毕在程序里就可以调用了，
	MemcachedUtil.put(key,value);
	MemcachedUtil.get(key,Object);
## 查询是否写入到缓存

	首先打开cmd，执行telnet命令，可以百度如何开启telnet功能
		telnet 192.168.201.140 11211
	上面这条命令可以连接上服务端的memcached,连接上之后可以通过查询是否插入值成功
		get key

更多命令可以查询百度


