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
MapReduce孕育了Hadoop，GFS孕育了HDFS，Bigtable孕育了Hbase。而在这三篇论文里面，经常提到一个协调
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
在讲解上述代码的流程前，先来补充一个小知识：
Zookeeper运用一个类似于linux的filesystem的系统，但他并没有文件和文件夹的概念，统称为Znode，
它既能作为一个容器来存储数据，也可以和其他Znode形成父子关系。
Znode有四种：`PERSISTENT`，`PERSISTENT_SEQUENTIAL`,`EPHEMERAL`,`EPHEMERAL_SEQUENTIAL`，也就是持久/暂时，顺序/非顺序的Znode的排列组合。

+ 持久/非持久节点：持久节点就是节点在创建后必须要主动调用delete来删除，而非持久节点如果客户端的会话失效，那么这个节点就会被主动清除。
+ 顺序节点：每个父节点会为他的第一级子节点维护一份时序，会记录每个子节点创建的先后顺序。基于这个特性，在创建子节点的时候，可以设置这个属性，那么在创建节点过程中，
ZK会自动为给定节点名加上一个数字后缀，作为新的节点名。

在Zookeeper的分布式锁实现里面，我们用到的就是EPHEMERAL_SEQUENTIAL（非持久顺序锁），这样一来保证了就算会话失败了，锁也可以被快速释放，二来也保证了最先创建Znode的子节点的会话拥有优先读写的权利。

讲完了Znode的小知识，我们来阐述一下上述这段代码的流程：
1. 首先我们创建想要获取锁的资源的Znode的子节点
2. 获取当前该Znode的所有子节点
3. 如果我们所创建的子节点是第一个节点（基于`SEQUENTIAL`），那么获取锁，执行操作，删除锁
4. 如果无法获取锁，那么注册一个watcher，用来监听所有子节点的变化，直到自己的回合，执行步骤3

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
Curator是一个基于Zookeeper API的实现，提供了很多可用的轮子，推荐使用，毕竟自己实现ZK的分布式锁还是有点复杂的。
