---
layout:     post
title:      "Docker容器里面的OOM问题"
subtitle:   " \"Running a JVM in a Container Without Getting OOM\""
date:       2018-07-31 15:42:00
author:     "Mao"
header-img: "img/home-bg.jpg"
comments: true
tags:
    - Java
    - JVM
    - Docker
---

> “呵 第二篇博客 过了八个月. ”


## 前言

前阵子组里面有个APP 一直OOM，基本上两三天OOM一次的节奏，devops受不了了，就给我们开了个ticket
，趁着这个机会看了下JVM 的一些option，最后问题也算顺利解决。

---

## 正文

观察heapdump的时候发现了一个很有趣的事情，明明一个container只给了2个G的内存但是，heapsize
经常是超过2个G的，这究竟为什么呢？

以前看一个帖子我记得有说过，在一个container里面运行一个JVM，它初始的HEAP maxsize 是1/4 的host
内存，而不是container本身的，为了验证这个想法我做一个简单的尝试：

```
jzhang@CPD-jzhang:~$ docker run -m 100MB openjdk:8u121 java -XshowSettings:vm
WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.
VM settings:
    Max. Heap Size (Estimated): 3.46G
    Ergonomics Machine Class: server
    Using VM: OpenJDK 64-Bit Server VM

```

你看 我运行了一个100MB的container，但是最后的heapsize却是我电脑内存的1/4,这不OOM都不科学了。
那么我们该怎么做才能避免这种情况呢？
因为组里面用的仍旧是JAVA8,JAVA8里面有个一个option可以让JVM 看到container本身拥有的内存：
`-XX:+UseCGroupMemoryLimitForHeap`

```
jzhang@CPD-jzhang:~$ docker run -m 1GB openjdk:8u131 java   -XX:+UnlockExperimentalVMOptions   -XX:+UseCGroupMemoryLimitForHeap   -XshowSettings:vm -version
WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.
VM settings:
    Max. Heap Size (Estimated): 228.00M
    Ergonomics Machine Class: server
    Using VM: OpenJDK 64-Bit Server VM

```
现在看起来就正常多了，我运行了一个1个G的container，然后他的最大的heapsize是200+MB（大约1/4吧）
但是问题来了，我的container有一个G，但我只用了这么点给heap是不是有点浪费啊？怎么办呢？
这时候我们可以使用： `-XX:MaxRAMFraction`
他会告诉JVM 去使用 `memory of container / MaxRAMFraction` 作为HEAP的大小，你可以根据自己的
情况来定义这个数字，然后1的时候当然就是100%使用所有可用的container内存啦。


最后插一句嘴，JAVA10 之后已经修改了这个问题，默认JVM 可以看到container的大小了。所以，你看
适时升级JDK version 还是很有必要的。

——Mao 后记于 paris 2018.07
