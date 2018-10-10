---
layout:     post
title:      "使用Zookeeper实现分布式锁"
subtitle:   " \"Using Zookeeper to realise a distributed lock \""
date:       2018-10-10 13:18:00
author:     "Mao"
header-img: "img/post-bg-2018-08.png"
comments: true
tags:
    - Zookeeper
    - 分布式
---

> “来，今天我们用zookeeper来实现一个分布式的锁。”


## 前言
在单机环境下，我们可以用JDK自带的API很轻松的来应对高并发的场景，但是这些API在分布式的场景里面就有些
有心无力了。这时候，我们就需要利用分布式锁来实现数据同步的问题，实现分布式锁的方式有很多，比如你可以用
数据库，你也可以用缓存。因为组里面推荐使用zookeeper，今天我们来讲一下如何用ZK 来实现分布式锁。

## 正文
### 一些关于Zookeeper 的简单介绍
也许有些人会不认识Zookeeper，但从事大数据这个行业的人对Google在2006年发布的三篇论文肯定耳熟能详，
MapReduce孕育了Hadoop,GFS孕育了HDFS，Bigtable孕育了Hbase。而在这三篇论文里面，经常提到一个协调
服务：Chubby，Zookeeper就是Chubby的一个开源实现。
那么，到底Zookeeper是什么呢？
>"ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. All of these kinds of services are used in some form or another by distributed applications."

这是Zookeeper官网的一个定义，我大概翻译一下就是：Zookeeper是一个中心化的协调服务，他用来解决分布式应用中经常会遇到的一些数据管理问题：比如统一配置信息，命名，
以及提供分布式的一些数据同步，集群管理问题。Zookeeper的实现基于ZAB协议（类似于PAXOS），篇幅有限就不讲了，有机会下次可以再写一篇blog。

### 使用Zookeeper 的API来实现分布式锁
```
ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
final String resource = "/resource";

final String lockNumber = zk
    .create("/resource/lock-", null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

List<String> locks = zk.getChildren(resource, false, null);
Collections.sort(locks);

if (lockNumber.endsWith(locks.get(0))) {
  System.out.println("Acquire Lock");
  zk.delete(lockNumber, 0);
} else {
  zk.getChildren(resource, new Watcher() {
    public void process(WatchedEvent watchedEvent) {
      try {
        ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
        List locks = zk.getChildren(resource, null, null);
        Collections.sort(locks);

        if (lockNumber.endsWith((String)locks.get(0))) {
          System.out.println("Acquire Lock");
          zk.delete(lockNumber, 0);
        }

      } catch (Exception e) {}
    }
  }, null);
}
```

### 使用Curator来实现基于Zookeeper 的分布式锁
```
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client = CuratorFrameworkFactory.newClient("host:ip",retryPolicy);
client.start();

InterProcessSemaphoreMutex lock = new InterProcessSemaphoreMutex(client, path);
if(lock.acquire(10, TimeUnit.SECONDS))
{
     try { /*do something*/ }
     finally { lock.release(); }
}
```
