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
AOP(Aspect Oriented Programming) 通俗的翻译过来就是切面编程，假设传统的OOP是一个二维平面，那么AOP就是在这些平面上加入了一个新的维度，
在不破坏现有平面的情况下，可以切入地增加一些新的功能，举个简单的例子，现在有几个做好的模块，然后你想加入一个验证记录的功能，传统的OOP么就是
在一个个模块里面建立日志记录的对象，这样不仅会侵入业务逻辑，产生冗余的代码，也不方便维护，这时候如果我们建立一个验证记录的切面，那么，在不改变原来模块的
情况下，我们就可以实现这个功能，是不是很COOL?!

## 正文
### 一个简单的消息发送 Service
假设我们有一个简单的消息发送功能，然后我们想要在消息发送前加入一个验证功能，以下是简化的代码：
```
public class MessageSender {
  public MessageSender() {
  }

  public static void sendMessage() {
    AuthUtil.auth();
    MessageUtil.sendMessage();
  }
}
```
这里，我们在主业务逻辑里面加入了验证功能，有没有办法把这部分从里面剥离出来呢？我们可以通过Proxy的
办法把这些具有横切性质的代码织如目标代码。

### JDK Proxy 实现
JDK Proxy 主要涉及了 `java.lang.reflect` 里面的两个类：InvocationHandler 和 Proxy。
前者是一个接口，我们可以通过实现该接口来定义具体织入的横切逻辑，后者通过反射来实现一个代理的实例。

在InvocationHandler只定义了一个方法：`public Object invoke(Object proxy, Method method, Object[] args)`
* proxy：最终生成的代理实例
* method：代理的类里面的某个方法
* args：method的传入参数

首先我们定义一个要被代理的对象：
```
public class JDKProxyMessageSender implements ExInterface {
  public void execute(){
    MessageUtil.sendMessage("JDK Proxy");
  }
}
```

接着，在不改动这部分代码的同时，让我们来创建代理类：
```
public class JDKProxy implements InvocationHandler {
  private JDKProxyMessageSender target;

  public JDKProxy(JDKProxyMessageSender target) {
    this.target = target;
  }

  public ExInterface createProxy() {
    return (ExInterface) Proxy
        .newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(),
            this);
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if ("execute".equals(method.getName())) {
      AuthUtil.auth();
      Object result = method.invoke(target, args);
      return result;
    }
    return method.invoke(target, args);
  }
}
```

代码测试：
```
//JDK Proxy
JDKProxyMessageSender a=new JDKProxyMessageSender();
JDKProxy jdkProxy=new JDKProxy(a);
ExInterface proxy=jdkProxy.createProxy();
proxy.execute();
System.out.println("\n");
```

输出：
```
Authentication processing...
Sending message by JDK Proxy
```

### CGLib 实现
代理类需要实现`MethodInterceptor`接口，然后通过`Enhancer`来创建代理类，`MethodIntercepto` 里面只有一个方法，intercept
类似于JDK Proxy 里面的invoke方法，用来描述织入目标方法的横切逻辑。

首先我们也需要定义一个需要被代理的对象，和JDK Proxy不同的是，这个对象不需要实现接口，
```
public class CGLibInstance {

  public void execute() {
    MessageUtil.sendMessage("CGLib Proxy");
  }
}
```

然后我们要创建代理类：
```
public class CGLibMessageSender implements MethodInterceptor {


  private CGLibInstance target;

  public CGLibMessageSender(CGLibInstance target) {
    super();
    this.target = target;
  }


  public CGLibInstance createProxy() {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(target.getClass());
    enhancer.setCallback(this);
    return (CGLibInstance) enhancer.create();
  }

  @Override
  public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy)
      throws Throwable {
    if ("execute".equals(method.getName())) {
      AuthUtil.auth();
      Object result = methodProxy.invokeSuper(proxy, args);
      return result;
    }
    return methodProxy.invokeSuper(proxy, args);
  }
}
```
代码实现：
```
//CGLib Proxy
CGLibInstance cgLibInstance=new CGLibInstance();
CGLibMessageSender cgLibProxy=new CGLibMessageSender(cgLibInstance);
cgLibInstance= cgLibProxy.createProxy();
cgLibInstance.execute();
```
输出：
```
Authentication processing...
Sending message by CGLib Proxy
```
### JDK Proxy和CGLib Proxy比较：
* 当你的目标对象实现了接口但调用的方法不是接口方法的时候，你只能使用CGLib proxy，因为JDK动态代理的原理就是实现和目标对象同样的接口，
因此只能调用那些接口方法。Cglib则没有此限制，因为它所创建出来的代理对象就是目标类的子类，因此可以调用目标类的任何方法。
* 就创建的动态代理对象的性能来说，CGLib 是 JDK 的 10 倍，而创建动态代理对象所花费的时间上，CGLib 却比 JDK 多花 8 倍的时间。
 所以，对于单例模式或者具有实例池的代理类，适合采用 CGLib 技术，反之，则适合采用 JDK 技术。

## 后记
SpringAOP会优先选择JDK动态代理，当调用方法不是接口方法时，就只能选择Cglib了，再下一篇blog我们会举一个例子，Spring究竟是怎么使用AOP的，本文的代码
也可以在我的[github](https://github.com/BravoMao/SpringAOPDemo)找到。
