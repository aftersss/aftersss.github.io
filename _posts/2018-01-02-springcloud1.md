---
layout: post
title:  "Spring Cloud实战（一）"
date:   2018-01-02
categories: technical
author: 黄文奇
---

> 本文介绍了spring-cloud使用过程中遇到的一些问题，以及解决方法。

我们公司最近刚转为使用spring cloud，在几个月内把大部分服务全面切换为spring cloud技术栈（之前使用的是dubbo），在过程中遇到了不少问题，这里对一些问题进行总结和回顾。

# 解决 connect timed out

某次有个新上线的调用大量抛出了java.net.SocketTimeoutException: connect timed out,于是我们开始着手分析，由于是内网调用，我们设置的连接超时时间为100ms，一开始我们怀疑是网络抖动导致的，于是把连接超时时间改为1秒，但是经过观察，情况并没有改善,所以基本可以排除是网络抖动的原因。

于是我们开始从其他角度分析，我们发现之前的其他系统之间的调用没有出现这个异常，唯独这次新上线的系统调用出现此异常，从qps来看，这个调用的qps是其他调用的qps的好几倍。结合服务提供方的gc日志来分析，我们发现连接超时异常在young gc的时候集中出现。于是我把怀疑的目光聚焦到了内核参数上。

我们先来分析下tcp的链接建立的策略：对于每个tcp监听端口，服务端维护着两个队列，一个是syn队列，一个是accept队列；当客户端发送syn握手消息给服务端之后，并且链接建立完成之前，这个链接会进入syn队列；当这个链接经过三次握手建立完成之后，链接会从syn队列移除，并转移到accept队列中；此时应用层方可从accept队列中取出tcp链接进行处理。

通过命令`cat /proc/sys/net/ipv4/tcp_max_syn_backlog`可以看到syn队列的最大长度，我们的服务器上该值为1024；

通过命令`cat /proc/sys/net/core/somaxconn`可以看到accept队列的最大长度，我们的服务器上该值为128；

我们来分析下conn timed out是怎么发生的：
1. 首先该服务的qps很高，当服务端发生young gc时，会导致调用方的已有tcp链接全部进入阻塞状态，导致调用方连接池没有可用资源(我们用的是apache的HttpClient)。（即使不发生young gc，由于调用qps高，仍然会出现此情况）
2. 客户端经过短暂的`空闲连接等待时间`后（我们用的是apache的HttpClient，并且设置了短暂的等待时间，这个等待时间用于在建立新链接之前先等待连接池有空闲资源），发现连接池仍然没有空闲资源，于是开始尝试建立新的tcp链接，注意此时会有大量线程同时新建tcp链接，新建tcp链接的过程在内核态发生，应用无需参与，并且此时由于服务端gc的发生，导致用户态不会从accept队列中拿走tcp链接，最终导致服务端128个坑位被占满，当accept队列被占满时，linux默认会忽略所有后续的握手请求（直接忽略tcp三次握手的最后一步的ack消息，导致tcp建链失败），从而导致客户端抛出连接超时异常。

对于这个问题我们有几种解法：
1. 优化young gc，我们服务端的young gc时间约为10-20ms，优化空间较小，不采用此方案。而且不是所有的连接超时都发生在young gc的时候。
2. 把http的`空闲连接等待时间`调大，这是个双刃剑，调的太大，影响rt，调的太小，tcp链接数会波动的更剧烈。我们目前配置的是50ms，可以不调整。
3. 把accept队列数调大。

综合评估后，我们选择了把accept数调大，在我们的例子中我们改为了1024，改完重启应用即可生效。经过一段时间观察，这个问题解决了。

> 实际上accept队列的大小除了受内核参数影响，还会受应用层设置的影响，java里新建Socket的时候有个backlog参数，端口的accept队列大小实际上取的是两者中更小的那个值，tomcat默认backlog为100，undertow默认为1000，你需要调整应用中的backlog到合适的值。

# 解决tcp链路不稳定的问题

服务上线后我们发现客户端和服务端之间的tcp链路不稳定，链接生命周期较短，一个刚建立好的tcp链接，往往几秒后就断开了，在内部调用中，我们应该让tcp链接保持尽量长时间的连接状态，以减少反复握手的开销，降低平均rt。
由于之前我们对spring-cloud调用使用了自定义的httpClient，按道理链接应该保持较长时间，是哪里出问题了？
经过调试，我们发现问题出在服务端: 我们目前在用的tomcat容器默认会在如下这些情况下断开链接：
1. 当一个http链接处理了100个请求后会强制断开链接，此参数可调优。
2. 当http响应状态码为500时会强制断开链接。

我们目前只发现了上面两种情况会断开链接，可能还有其他情况。我们暂时只优化了第一条，上线后效果较明显；后来我们发现undertow容器在这些情况下不会断开链接，默认配置下就可以保持稳定的http链接，于是我们把部分服务器的内嵌tomcat替换为了undertow。如何替换为undertow？网上的文章汗牛充栋，这里就不介绍了。

## 下期预告
下期我们将会探讨我们对spring-cloud的一些优化。

# 参考资料
关于tcp连接队列、半连接队列等知识，这篇文章讲的很好：
https://mp.weixin.qq.com/s/yH3PzGEFopbpA-jw4MythQ
