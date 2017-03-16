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

>首次连接会发现无法连接redis，因为redis有保护模式，在redis.conf里面去掉保护模式，然后启动的时候指定配置文件 src/redis-server redis.conf

	protected-mode no

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


## redis与Spring项目整合

>不同之处在于将缓冲池交给redis进行管理

	<!-- master连接池参数 -->
   	<bean id="masterPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
		<!-- <property name="maxActive" value="${redis.master.pool.max_active}"/> -->
		<property name="maxIdle" value="${redis.master.pool.max_idle}" />
		<property name="maxWaitMillis" value="${redis.master.pool.max_wait}" />
		<property name="testOnBorrow" value="${redis.master.pool.testOnBorrow}" />
		<property name="testOnReturn" value="${redis.master.pool.testOnReturn}" />
	</bean>
	
	<bean id="jedisPool" class="redis.clients.jedis.JedisPool"
		destroy-method="destroy">
		<constructor-arg index="0" ref="masterPoolConfig" />
		<constructor-arg index="1" value="${redis.master.server.ip}" />
		<constructor-arg index="2" value="${redis.master.server.port}" type="int" />
	</bean>

> 访问客户端,这里仅提供操作key的一个方法，还可以操作List,Hash,Set,String等

```java

@Service
public class JedisService {
	private static final int REDIS_DB_INDEX = 0;
	private static Logger LOG = LoggerFactory.getLogger(JedisService.class.getSimpleName());

	@Autowired
	private JedisPool jedisPool;

	private void saveAndReturnResource(Jedis jedis) {
		String result = jedis.save();
		LOG.debug("Redis数据保存结果:{}", result);
		returnResource(jedis);
	}

	private void returnResource(Jedis jedis){
		jedisPool.returnResource(jedis);
	}
	
	private Jedis getJedis(){
		Jedis jedis = jedisPool.getResource();
		String select = jedis.select(REDIS_DB_INDEX);
		LOG.debug("Redis {}号数据库选择结果:{}", REDIS_DB_INDEX, select);
		return jedis;
	}
	
	/** 
	 * 清空当前数据库 
	 * @return 状态码 
	 * */
	public String flushDB() {
		Jedis jedis = getJedis();
		String stata = jedis.flushDB();
		saveAndReturnResource(jedis);
		return stata;
	}
	
	/**操作Key的方法*/
	public Keys KEYS = new Keys();
	public class Keys {
		/** 
		 * 更改key 
		 * @param String oldkey 
		 * @param String newkey 
		 * @return 状态码 
		 * */
		public String rename(String oldkey, String newkey) {
			return rename(SafeEncoder.encode(oldkey), SafeEncoder.encode(newkey));
		}

		/** 
		 * 更改key,仅当新key不存在时才执行 
		 * @param String oldkey 
		 * @param String newkey 
		 * @return 状态码 
		 * */
		public long renamenx(String oldkey, String newkey) {
			Jedis jedis = getJedis();
			long status = jedis.renamenx(oldkey, newkey);
			saveAndReturnResource(jedis);
			return status;
		}

		/** 
		 * 更改key 
		 * @param String oldkey 
		 * @param String newkey 
		 * @return 状态码 
		 * */
		public String rename(byte[] oldkey, byte[] newkey) {
			Jedis jedis = getJedis();
			String status = jedis.rename(oldkey, newkey);
			saveAndReturnResource(jedis);
			return status;
		}

		/** 
		 * 设置key的过期时间，以秒为单位 
		 * @param String key 
		 * @param 时间,已秒为单位 
		 * @return 影响的记录数 
		 * */
		public long expired(String key, int seconds) {
			Jedis jedis = getJedis();
			long count = jedis.expire(key, seconds);
			returnResource(jedis);
			return count;
		}

		/** 
		 * 设置key的过期时间,它是距历元（即格林威治标准时间 1970 年 1 月 1 日的 00:00:00，格里高利历）的偏移量。 
		 * @param String key 
		 * @param 时间,已秒为单位 
		 * @return 影响的记录数 
		 * */
		public long expireAt(String key, long timestamp) {
			Jedis jedis = getJedis();
			long count = jedis.expireAt(key, timestamp);
			returnResource(jedis);
			return count;
		}

		/** 
		 * 查询key的过期时间 
		 * @param String key 
		 * @return 以秒为单位的时间表示 
		 * */
		public long ttl(String key) {
			Jedis jedis = getJedis();
			long len = jedis.ttl(key);
			returnResource(jedis);
			return len;
		}

		/** 
		 * 取消对key过期时间的设置 
		 *@param key 
		 *@return 影响的记录数 
		 * */
		public long persist(String key) {
			Jedis jedis = getJedis();
			long count = jedis.persist(key);
			returnResource(jedis);
			return count;
		}

		/** 
		 * 删除keys对应的记录,可以是多个key 
		 * @param String... keys 
		 * @return 删除的记录数 
		 * */
		public long del(String... keys) {
			Jedis jedis = getJedis();
			long count = jedis.del(keys);
			saveAndReturnResource(jedis);
			return count;
		}

		/** 
		 * 删除keys对应的记录,可以是多个key 
		 * @param String... keys 
		 * @return 删除的记录数 
		 * */
		public long del(byte[]... keys) {
			Jedis jedis = getJedis();
			long count = jedis.del(keys);
			saveAndReturnResource(jedis);
			return count;
		}

		/** 
		 * 判断key是否存在 
		 * @param String key 
		 * @return boolean 
		 * */
		public boolean exists(String key) {
			Jedis jedis = getJedis();
			boolean exis = jedis.exists(key);
			returnResource(jedis);
			return exis;
		}

		/** 
		 * 对List,Set,SortSet进行排序,如果集合数据较大应避免使用这个方法 
		 * @param String key 
		 * @return List<String> 集合的全部记录 
		 * **/
		public List<String> sort(String key) {
			Jedis jedis = getJedis();
			List<String> list = jedis.sort(key);
			saveAndReturnResource(jedis);
			return list;
		}

		/** 
		 * 对List,Set,SortSet进行排序或limit 
		 * @param String key 
		 * @param SortingParams parame 定义排序类型或limit的起止位置. 
		 * @return List<String> 全部或部分记录 
		 * **/
		public List<String> sort(String key, SortingParams parame) {
			Jedis jedis = getJedis();
			List<String> list = jedis.sort(key, parame);
			saveAndReturnResource(jedis);
			return list;
		}

		/** 
		 * 返回指定key存储的类型 
		 * @param String key 
		 * @return String  string|list|set|zset|hash 
		 * **/
		public String type(String key) {
			Jedis jedis = getJedis();
			String type = jedis.type(key);
			returnResource(jedis);
			return type;
		}

		/** 
		 * 查找所有匹配给定的模式的键 
		 * @param String key的表达式,*表示多个，？表示一个 
		 * */
		public Set<String> kyes(String pattern) {
			Jedis jedis = getJedis();
			Set<String> set = jedis.keys(pattern);
			returnResource(jedis);
			return set;
		}
	}

}
```

> redis不提供直接存储实体类的功能，当时可以先把实体类转为bytes[]，然后存入，取得时候再反转一下，下面是工具类：

```java
public class SerializeUtil {
	public static byte[] serialize(Object object) {
		ObjectOutputStream oos = null;
		ByteArrayOutputStream baos = null;
		try {
			//序列化  
			baos = new ByteArrayOutputStream();
			oos = new ObjectOutputStream(baos);
			oos.writeObject(object);
			byte[] bytes = baos.toByteArray();
			return bytes;
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	public static Object unserialize(byte[] bytes) {
		ByteArrayInputStream bais = null;
		try {
			//反序列化  
			bais = new ByteArrayInputStream(bytes);
			ObjectInputStream ois = new ObjectInputStream(bais);
			return ois.readObject();
		} catch (Exception e) {

		}
		return null;
	}
}

```

## 结束

	到这里如何在项目里使用redis就结束了，欢迎讨论！