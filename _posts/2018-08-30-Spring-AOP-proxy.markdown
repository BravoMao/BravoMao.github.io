---
layout:     post
title:      "Spring AOP 原理（JDK和CGLib 代理）"
subtitle:   " \"JDK and CGLib proxy \""
date:       2018-08-30 20:42:00
author:     "Mao"
header-img: "img/post-bg-2018-08.png"
comments: true
tags:
    - JAVA
    - CGLib
    - Proxy
---

> “来 今天我们来讲一下 Spring AOP的底层实现原理. ”


## 前言


---

## 正文
### 一个简单的Service
AOP(Aspect Oriented Programming) 通俗的翻译过来就是切面编程，假设传统的OOP是一个二维平面，那么AOP就是在这些平面上加入了一个新的维度，
在不破坏现有平面的情况下，可以切入地增加一些新的功能，举个简单的例子，现在有几个做好的模块，然后你想加入一个日志记录的功能，传统的OOP么就是
在一个个模块里面建立日志记录的对象，这样不仅会有很多冗余的代码，也不方便维护，这时候如果我们建立一个日志记录的切面，那么，在不改变原来模块的
情况下，我们就可以实现这个功能，是不是很COOL?!

### JDK Proxy 实现
### CGLib 实现
