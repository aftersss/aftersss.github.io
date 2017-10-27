---
layout: post
title:  "Redis序列化机制解析"
date:   2017-10-27 11:06:12 +0800
categories: technical
author: 黄文奇
keywords: redis, memcached, 序列化
description: 本文分析了Redis序列化机制，以及与memcached序列化机制的区别。
---

> 本文分析了Redis序列化机制，以及与memcached序列化机制的区别。

# redis序列化机制分析
最近经常有开发来询问我string-data-redis框架封装过的`StringRedisTemplate`和`RedisTemplate`有什么区别，spring为什么要提供这两个封装？为什么与我们自行封装的RedisCacheClient不能兼容？
在回答这些问题之前，我们先来看下RedisTemplate的用法：

```
RedisTemplate<String, User> redisTemplate = initIt....;
redisTemplate.opsForValue().get("id1");//从redis获取id为1的用户
```

我们再来看下StringRedisTemplate的源码片段：

```
public class StringRedisTemplate extends RedisTemplate<String, String> {
    ...
}
```

初看起来StringRedisTemplate只是继承了RedisTemplate<String, String>，那为什么我们不能直接用RedisTemplate<String, String>呢？表面上来看RedisTemplate已经完全可以满足需求了，为什么还要有个StringRedisTemplate?
其实答案已经隐藏在了源码里，如果你再深入一些，就会发现RedisTemplate默认使用的序列化方式和StringRedisTemplate是不同的：
我们先来看RedisTemplate使用的默认序列化方式：

```
public void afterPropertiesSet() {

    super.afterPropertiesSet();

    boolean defaultUsed = false;

    if (defaultSerializer == null) {

        // RedisTemplate默认使用了JdkSerializationRedisSerializer序列化方式。
        defaultSerializer = new JdkSerializationRedisSerializer(
                classLoader != null ? classLoader : this.getClass().getClassLoader());
    }
		...
}
```

我们再来看看StringRedisTemplate的源码：

```
public StringRedisTemplate() {
    //StringRedisTemplate使用StringRedisSerializer覆盖了默认序列化方式
    RedisSerializer<String> stringSerializer = new StringRedisSerializer();
    setKeySerializer(stringSerializer);
    setValueSerializer(stringSerializer);
    setHashKeySerializer(stringSerializer);
    setHashValueSerializer(stringSerializer);
}
```

从上述代码我们可以看出：
RedisTemplate默认使用jdk序列化方式，就是说RedisTemplate对于你的key和value都会使用jdk序列化方式转换成byte数组，然后再调用redis api；
而StringRedisTemplate默认使用String序列化方式，就是说对于你的key和value，它会使用string.getBytes(charset)转换成byte数组，然后再调用redis api；

他们的默认序列化方式完全不同，所以这两个Template是不兼容的，就是说如果你用RedisTemplate设置了一个key，用StringRedisTemplate是拿不出来的。对于同一个key，你永远应该使用同一个Template来操作，否则可能出现各种问题。
特殊地，如果你要调用redis计数器相关api，那么你只应该使用StringRedisTemplate。

而我们自己实现的RedisCacheClient也使用了跟上面不一样的序列化方式，所以跟这些Template也是不兼容的。


## memcached序列化机制分析
了解了以上这些内容，有些也用过memcached的同学可能会提出一个问题：
为什么memcached的api没有这么复杂？

我们先来看下典型的xmemcached客户端的set api设计：

```
public boolean set(final String key, final int exp, final Object value)
			throws TimeoutException, InterruptedException, MemcachedException;
```

对比下类似的客户端jedis set api的设计：

```
public boolean set(String key, String value);

public boolean set(byte[] key, byte[] value);
```

你可以看到memcached的api支持直接设置Object类型的value，而redis只能设置String或byte[]类型的value（事实上即使你用了String类型的api，jedis内部也会把它转成byte[]）
jedis这样设计就是告诉我们，它不关心序列化方式，它把这个问题抛给使用者来解决（幸好spring提供了易用性更好的api封装）。

从易用性来说，如果jedis能直接提供允许设置Object类型的value岂不更好，为什么它没有提供呢？
要理解这个问题，我们先假设jedis提供了下面这样的api,并且jedis在内部对于value使用了jdk序列化方式：

```
public boolean set(String key, Object value);
```

看起来好像没毛病，但实际上问题大了，考虑下面这样的调用方式：

```
jedis.set("count", 10);
jedis.incr("count", 1);
```

以上调用先把count初始化为10，然后试图对它进行自增，这里第二步会出错。
为什么呢？我们先看第一步，由于采用了jdk序列化方式，所以序列化的结果会带上对象类型标记，比如10的序列化结果可能是:`int10`,这里的int前缀表示接下来的数据是int类型的数据，如果不带上这个标记，jdk反序列化时就无法判断数据类型。
好了，第一步执行完后，redis服务器里的value是int10，第二步incr的时候，redis服务器会发现这个value压根不是一个有效的数字（redis服务器是跨语言的，它才不知道你用的是什么序列化方式），当然会报错了。

用过memcached api的同学都知道，上述先set后incr的调用对于xmemcached是行的通的，从底层来说，memcached服务器内部存储的数据也是字节数组，从而xmemcached客户端也必定要使用某种序列化方式把value转成字节数组，为了后续的反序列化，value字节数组里应该也有数据类型的标记，大家可能会疑惑，为什么memcached行得通呢？
我们可以从xmemcached的源码里看出端倪（这里就不贴了），简单来说，memcached的value的前几个byte是用于存储元数据的，元数据之后才是真正的有效数据，xmemcached就是利用这几个byte来做文章，它把当前数据类型标记放在了元数据中；比如你放的value是一个int，那么xmemcached会在元数据中标记这是个int，当你后续对它进行自增时，memcached服务器能根据元数据识别到这个value是个int，可以自增（memcached的元数据还有其他用处，比如标记value是否是经过压缩的，等等）。
正是因为元数据的存在，xmemcached的api才可以做到这么易用。
当然元数据也是有开销的，由于每个value都带有元数据，必定增加了服务器的内存压力和网络传输压力。redis或许是考虑到这点，为了数据的精简而没有支持元数据，但却为不少开发者带来了困惑。

## 结论
为了避免大家走进误区，我在文末再强调一遍：
对于同一个redis的key，请始终使用同一个redis客户端来操作（除非你能确保不同客户端的序列化方式是一致的）！

