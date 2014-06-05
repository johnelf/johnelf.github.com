---
layout: post
title: "数据库连接池的超时问题——BoneCP"
category: Technical
tags: ["连接池","超时","资源分配"]
---

## 连接池是什么

当在使用数据库做数据持久化时，我们需要通过数据库驱动来让程序与数据库打交道，比如esql和jdbc，这样在需要做CRUD操作时，我们只需要通过驱动建立连接，通过连接句柄操作SQL，在完成操作后再次释放连接。  但如果每次操作SQL时都需要申请新的连接、处理超时等候、释放连接，那整个处理的过程就会被拖慢，以至于成为性能瓶颈，设计模式中有一个资源池模式，相应的我们可以创建一个数据库连接池。
连接池的思想是用户不需要自己管理连接，只需要在application启动的时候初始化连接池然后在需要的地方直接从连接池去拿就行，连接池负责维护应用到数据库的连接，维护连接的保持、重建和回收。

## 连接池的原理
在系统启动时连接池首先建立最少可用的连接数量，然后每次在需要的时候将可用的连接分配给相应的请求，在应用使用完后资源池负责回收连接，并重新将该连接放回可用队列中。

## 高性能连接池代表BoneCP
BoneCP采用actor模型，由于是异步操作，所有结果用Future返回，超时返回，除去了同步等待的时间，效率极高。BoneCP的BenchMark就不贴了，想看的同学可以移步这里：[BoneCP 官网](http://www.jolbox.com/)  

  * The tests were run under Linux running in single user mode i.e. apart from the kernel daemons there were no other running processes.  
  * Used the latest, publicly available, stable versions of C3P0/BoneCP/DBCP/etc.  
  * Ran each test 3 times and took averages.  
  * Started off with a dummy connection request to allow for lazy-loading hits, JIT-compiler to kick in, etc.  
  * Gave the same parameters to each pool. Since there are no partitioning options in C3P0/DBCP, allocated (connections / partitionCount) to each partition (i.e. kept the same number of connections). A BoneCP partition count of 1 is thus the equivalent of running C3P0/DBCP.  
  * Called the getConnection/releaseConnection method calls directly (i.e. not obtaining it via spring/hibernate)  

每当向BoneCP申请一个Connection资源时：  

1. 判断自己是否关闭（即调用了shutdown方法），如果发现关闭，就抛出SQLException  
2. 获取当前线程ID，按分区数量对该值取模，计算出要访问的分区数组下标。  
3. 按下标获取分区对象（ConnectionPartition），然后从分区持有的队列中poll出一个空闲的connection对象（类型为ConnectionHandle，它包装了JDBC connection）。  
4. 如果poll的结果为null，说明该分区的队列中没有空闲的connection，那么从分区数组的0号开始轮询每个分区，直到poll出一个非null的connection。（循环结束之后仍有可能为null）  
5. 判断分区的connection是否达到了上限，如果没有，就向队列中插入一个新的connection  
6. 如果得到的结果还是为null，那么就调用分区的“阻塞一定时间的poll方法”，当该poll方法执行结束，仍然为null，就抛出SQLException  
7. 调用ConnectionHandle对象的renewConnection方法，将该对象标记为“打开（也就是设置一个boolean变量）”。这一方法会将当前线程设置进ConnectionHandle对象。  
8. 判断该ConnectionHandle对象是否持有一个Hook，如果有，就执行Hook的onCheckOut方法，并将该ConnectionHandle对象作为参数传入。Hook对象用以监听connection的生命周期。  
9. 如果配置了closeConnectionWatch属性为true，就开启一个线程来监听ConnectionHandle对象的close操作。这是一种用于debug的功能。  
10. 最后将该ConnectionHandle对象作为java.sql.Connection类型返回。

BoneCP的各项配置参数如下所示：

    partitionCount: 分区数
    maxConnectionsPerPartition: 分区最大连接数
    minConnectionsPerPartition: 分区最小连接数
    acquireIncrement: 每次增加的连接数
    acquireRetryAttempts: 重试次数
    acquireRetryDelay: 重试等候时间
    connectionTimeout: 连接超时时间
    maxConnectionAge: 连接最大存活时间
    idleMaxAge: 连接最大空闲时间
    idleConnectionTestPeriod: 连接空闲测试周期
    disableJMX: JMX的开关
    statisticsEnabled: 连接池统计信息开关
    disableConnectionTracking: 是否记录连接的开关
    queryExecuteTimeLimit: SQL执行的时间限制

## BoneCP的超时问题
虽然BoneCP的性能很强劲，但最近在项目上遇到一个关于connection pooling的问题，BoneCP在长时间使用后会时而抛出Timed Out: waiting for a free available connection的SQLException。
经过几天的分析，排除了数据库端最大连接数的问题，发现问题可能出现在释放连接资源时，从正在使用或者正在处理的队列回收的连接没有被正常回收到available queue中，也就是说回收与可用之间消息的同步出现了问题。  
但这种可能无法验证，因为得到的始终是表象，无法知道是不是同步出现了问题，而且目前也不大清楚这种同步机制是如何实现的。  
最后一种可能就是在清理空闲进程或者杀死已经存活某段时间的连接时，这些被清理的连接会从available queue出队从而有可能导致waiting for a available connection timed out。但由于问题出现在产品环境，所以排除idle的情况，只考虑maxConnectionAge，这个参数会清理存活够特定时间的连接，从而可能导致超时。
所以最后的解决方案就是将maxConnectionAge的值设为0，目前仍在观察中。

{% include JB/setup %}
