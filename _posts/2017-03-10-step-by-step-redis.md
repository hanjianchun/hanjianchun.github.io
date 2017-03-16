---
layout: post
title:  "一步一步开始使用redis缓存"
categories: redis
tags:  redis
author: hanjianchun
---

* content
{:toc}

从安装redis开始到使用redis，一步一步教你怎么使用redis作为项目的缓存。



## 安装redis

>首先到[https://redis.io/download](https://redis.io/download "redis官网")去下载最新的redis*.tar.gz,我下载的是3.2.8版本。

>将redis放到某个文件夹里然后运行命令

	$ tar xzf redis-3.2.8.tar.gz
	$ cd redis-3.2.8
	$ make && make install

>到这里就安装完成了！之后启动redis server

	$ src/redis-server

>使用命令查看redis服务是否启动

	$ ps -ef | grep redis

>使用客户端验证是否安装成功：

	$ src/redis-cli
	redis> set foo bar
	OK
	redis> get foo
	"bar"

## 在项目里使用redis

>需要redis的jar包，去maven仓库搜索jedis就会出现，使用jedis作为redis的客户端

	<!-- https://mvnrepository.com/artifact/redis.clients/jedis -->
	<dependency>
	    <groupId>redis.clients</groupId>
	    <artifactId>jedis</artifactId>
	    <version>2.9.0</version>
	</dependency>

> 1.新建redis.properties作为自己的配置文件，里面存储redis的一些连接配置信息

	redis.master.server.ip=10.20.70.89
	redis.master.server.port=6379
	redis.master.pool.max_active=50
	redis.master.pool.max_idle=5
	redis.master.pool.max_wait=1000
	redis.master.pool.testOnBorrow=true
	redis.master.pool.testOnReturn=true

> 2.新建redis客户端jedis缓冲池类 RedisClientPool.java

```java
public class RedisClientPool {

	public static JedisPool pool;

	public static JedisPool getPool() {
		if (pool == null) {
			ResourceBundle bundle = ResourceBundle.getBundle("redis");
			if (bundle == null) {
				throw new IllegalArgumentException("[redis.properties] is not found!");
			}
			JedisPoolConfig config = new JedisPoolConfig();
			
			config.setMaxTotal(Integer.valueOf(bundle.getString("redis.master.pool.max_active")));
			config.setMaxIdle(Integer.valueOf(bundle.getString("redis.master.pool.max_idle")));
			config.setMaxWaitMillis(Long.valueOf(bundle.getString("redis.master.pool.max_wait")));
			config.setTestOnBorrow(Boolean.valueOf(bundle.getString("redis.master.pool.testOnBorrow")));
			config.setTestOnReturn(Boolean.valueOf(bundle.getString("redis.master.pool.testOnReturn")));
			pool = new JedisPool(config, bundle.getString("redis.master.server.ip"), Integer.valueOf(bundle.getString("redis.master.server.port")));
		}
		return pool;
	}
}

```

> 3.新建一个redis的工具类，操作缓存 RedisClientUtils.java

```java
public class RedisClientUtils {

	/**
	 * 判断该键是否存在，存在返回1，否则返回0
	 * 
	 * @param key
	 * @return
	 */
	public static boolean exists(final String key) {
		Jedis jedis = RedisClientPool.getPool().getResource();
		boolean bool = jedis.exists(key);
		RedisClientPool.getPool().returnResource(jedis);
		return bool;
	}

	/**
	 * 设定该Key持有指定的字符串Value，如果该Key已经存在，则覆盖其原有值。
	 * 
	 * @param key
	 * @param value
	 */
	public static void set(final String key, final String value) {
		Jedis jedis = RedisClientPool.getPool().getResource();
		jedis.set(key, value);
		RedisClientPool.getPool().returnResource(jedis);
	}

	/**
	 * 获取指定Key的Value，如果该Key不存在，返回null。
	 * 
	 * @param key
	 * @return
	 */
	public static String get(final String key) {
		Jedis jedis = RedisClientPool.getPool().getResource();
		String str = jedis.get(key);
		RedisClientPool.getPool().returnResource(jedis);
		return str;
	}

	/**
	 * 删除指定的Key
	 * 
	 * @param keys
	 * @return
	 */
	public static long delete(final String... keys) {
		Jedis jedis = RedisClientPool.getPool().getResource();
		long l = jedis.del(keys);
		RedisClientPool.getPool().returnResource(jedis);
		return l;
	}

	/**
	 * 命名指定的Key, 如果参数中的两个Keys的命令相同，或者是源Key不存在，该命令都会返回相关的错误信息。如果newKey已经存在，则直接覆盖。
	 * 
	 * @param oldkey
	 * @param newkey
	 */
	public static void rename(final String oldkey, final String newkey) {
		Jedis jedis = RedisClientPool.getPool().getResource();
		jedis.rename(oldkey, newkey);
		RedisClientPool.getPool().returnResource(jedis);
	}

	/**
	 * 设置某个key的过期时间（单位：秒）, 在超过该时间后，Key被自动的删除。如果该Key在超时之前被修改，与该键关联的超时将被移除。
	 * 
	 * @param key
	 * @param seconds
	 * @return
	 */
	public static void setTime(final String key, final int seconds) {
		Jedis jedis = RedisClientPool.getPool().getResource();
		jedis.expire(key, seconds);
		RedisClientPool.getPool().returnResource(jedis);
	}

	/**
	 * 在指定Key所关联的List
	 * Value的头部插入参数中给出的所有Values。如果该Key不存在，该命令将在插入之前创建一个与该Key关联的空链表
	 * ，之后再将数据从链表的头部插入。如果该键的Value不是链表类型，该命令将返回相关的错误信息。
	 * 
	 * @param key
	 * @param obj
	 * @return
	 */
	public static boolean leftPush(String key, Object obj) {
		if (StringUtils.isBlank(key) || obj == null) {
			return false;
		}
		Jedis jedis = RedisClientPool.getPool().getResource();
		jedis.lpush(SerializeUtil.serialize(key), SerializeUtil.serialize(obj));
		RedisClientPool.getPool().returnResource(jedis);
		return true;
	}

	/**
	 * 存入list对象,不覆盖之前在原来的key基础上存
	 * 
	 * @param key
	 * @param objList
	 * @return
	 */
	public static boolean listPush(String key, List<Object> objList) {
		if (StringUtils.isBlank(key) || objList == null) {
			return false;
		}
		Jedis jedis = RedisClientPool.getPool().getResource();
		for (Object object : objList) {
			if (object != null) {
				jedis.lpush(SerializeUtil.serialize(key), SerializeUtil.serialize(object));
			}
		}
		RedisClientPool.getPool().returnResource(jedis);
		return true;
	}

	/**
	 * 存入object数组对象,不覆盖之前在原来的key基础上存
	 * 
	 * @param key
	 * @param objs
	 * @return
	 */
	public static boolean listPush(String key, Object... objs) {
		if (StringUtils.isBlank(key) || objs.length <= 0) {
			return false;
		}
		Jedis jedis = RedisClientPool.getPool().getResource();
		for (Object obj : objs) {
			if (obj != null) {
				jedis.lpush(SerializeUtil.serialize(key), SerializeUtil.serialize(obj));
			}
		}
		RedisClientPool.getPool().returnResource(jedis);
		return true;
	}

	/**
	 * 在指定Key所关联的List
	 * Value的尾部插入参数中给出的所有Values。如果该Key不存在，该命令将在插入之前创建一个与该Key关联的空链表
	 * ，之后再将数据从链表的尾部插入。如果该键的Value不是链表类型，该命令将返回相关的错误信息。
	 * 
	 * @param key
	 * @param obj
	 * @return
	 */
	public static boolean rightPush(String key, Object obj) {
		if (StringUtils.isBlank(key) || obj == null) {
			return false;
		}
		Jedis jedis = RedisClientPool.getPool().getResource();
		jedis.rpush(SerializeUtil.serialize(key), SerializeUtil.serialize(obj));
		RedisClientPool.getPool().returnResource(jedis);
		return true;
	}

	/**
	 * 查询list类型的缓存数据
	 * 
	 * @param key
	 * @return
	 */
	public static List<Object> getListValueByKey(String key) {
		Jedis jedis = RedisClientPool.getPool().getResource();
		List<Object> valueList = new ArrayList<Object>();
		long size = jedis.llen(SerializeUtil.serialize(key));
		if (size > 0) {
			List<byte[]> tempList = jedis.lrange(SerializeUtil.serialize(key), 0, -1);
			if (tempList != null) {
				for (byte[] temp : tempList) {
					Object obj = SerializeUtil.unserialize(temp);
					valueList.add(obj);
				}
			}
		}
		RedisClientPool.getPool().returnResource(jedis);
		return valueList;
	}

	/**
	 * 根据条件删除缓存队列
	 * 
	 * @param key
	 */
	public static void removeListTypeRedisByKey(String key) {
		Jedis jedis = RedisClientPool.getPool().getResource();
		long size = jedis.llen(SerializeUtil.serialize(key));
		if (size > 0) {
			for (int i = 0; i < size; i++) {
				jedis.lpop(SerializeUtil.serialize(key));
			}
		}
		RedisClientPool.getPool().returnResource(jedis);
	}
}
```

> 4.有了这些后就可以开始测试缓存工具类的使用
	
	//放数据
	RedisClientUtils.set("han", "value");
	//取数据
	RedisClientUtils.get("han");

其它的方法需要自己去摸索尝试