---
layout:     post
title:      "Spring AOP 整理"
subtitle:   " \"A simple spring AOP tutorial \""
date:       2018-08-21 14:42:00
author:     "Mao"
header-img: "img/post-bg-2018-08.png"
comments: true
tags:
    - Java
    - Spring
    - AOP
---

> “来 今天我们来讲一下 Spring AOP. ”


## 前言
说实话用了AOP这个概念出来蛮久了，很多人把它放在OOP的对立面做比较，其实真的没必要，在我看来AOP更像是OOP的一种补充，
他帮助解决了OOP中代码冗余的问题，使工程师能加致力于解决问题本身，对于其他诸如日志啊，验证啊，和主逻辑无关的部分可以交给AOP，
这样写成出来的代码更精简也更易读，AOP 的实现有好几种比如AspectJ，SpringAOP，这里我们只讨论SpringAOP。
两者的比较可[参考](https://www.baeldung.com/spring-aop-vs-aspectj)。

---

## 正文
### AOP 究竟是什麼
AOP(Aspect Oriented Programming) 通俗的翻译过来就是切面编程，假设传统的OOP是一个二维平面，那么AOP就是在这些平面上加入了一个新的维度，
在不破坏现有平面的情况下，可以切入地增加一些新的功能，举个简单的例子，现在有几个做好的模块，然后你想加入一个日志记录的功能，传统的OOP么就是
在一个个模块里面建立日志记录的对象，这样不仅会有很多冗余的代码，也不方便维护，这时候如果我们建立一个日志记录的切面，那么，在不改变原来模块的
情况下，我们就可以实现这个功能，是不是很COOL?!

### SpringAOP 的一些基本概念
#### Joinpoint
> a point during the execution of a program, such as the execution of a method or the handling of an exception

连接点，定义了程序执行的时候切面织入的点，是一个具体的点，比如某个方法。
#### Pointcut
> a predicate that matches join points

切点，相比于连接点，切点更抽象一点，更宏观一点，描述了某一类joint points在何处执行，比如：
```
@Pointcut("execution(* * doSomething())")
  public void somePointCut() {}
```
#### Advice
> action taken by an aspect at a particular join point

通知，定义了在何时要执行想要织入的逻辑,大概有以下几类：
* Before advice
* After throwing advice
* After returning advice
* Around advice

举个例子：
```
@Component
@Aspect
public class LoggingAspect {

    private Logger logger = Logger.getLogger(LoggingAspect.class.getName());

    @Pointcut("@target(org.springframework.stereotype.Repository)")
    public void repositoryMethods() {};

    @Before("repositoryMethods()")
    public void logMethodCall(JoinPoint jp) {
        String methodName = jp.getSignature().getName();
        logger.info("Before " + methodName);
    }
}
```
在上面这个例子里面，logMethodCall会在所有Repository的方法前执行。

#### Aspect
> a modularization of a concern that cuts across multiple classes

切面，切面是切点，通知的集合体，他定义了如何将切点们织入到指定的连接点中，也定义了织入的代码逻辑。

### SpringAOP 实现原理

### SpringAOP 的一个基本应用
