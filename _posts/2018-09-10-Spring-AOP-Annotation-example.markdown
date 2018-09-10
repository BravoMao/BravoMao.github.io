---
layout:     post
title:      "Spring AOP 简单应用"
subtitle:   " \"A custom annotation implemented by Spring AOP \""
date:       2018-09-10 17:42:00
author:     "Mao"
header-img: "img/post-bg-2018-08.png"
comments: true
tags:
    - Annotation
    - Spring
    - Spring AOP
---

> “来 今天我们用Spring AOP实现一个定制的Spring annotation。”


## 前言
还记得上一篇里面我们用动态代理的方法实现了在发送消息之前加入了一个横切的验证的方法么？这一次，我们用
SpringAOP Annotation再来实现一遍。

## 正文
### 定义一个Authentication的annotation
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Authentication {

}
```
### 定义一个Aspect，以及切点和通知
这里，我们增加了两个通知，在连接点前加入一个验证的方法，在方法执行前后，记录执行时间
```
@Aspect
@Component
public class ExampleAspect {
  @Before("@annotation(Authentication)")
  public void authentication() {
    System.out.println("Authentication processing...");
  }

  @Around("@annotation(Authentication)")
  public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
    long start = System.currentTimeMillis();

    Object proceed = joinPoint.proceed();

    long executionTime = System.currentTimeMillis() - start;

    System.out.println(joinPoint.getSignature() + " executed in " + executionTime + "ms");
    return proceed;
  }
}
```

### 定义一个jointpoint（连接点）

```
@Authentication
public void sendMessage() throws InterruptedException {
  Thread.sleep(2000);
}
```

### 输出
```
Authentication processing...
void com.jianli.demo.aop.MessageSender.sendMessage() executed in 2012ms
```

## 后记
本文的代码也可以在我的[github](https://github.com/BravoMao/SpringAOPDemo)找到。
