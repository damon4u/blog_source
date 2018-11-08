---
title: Spring AOP类内部方法调用不拦截
date: 2018-11-08 15:41:50
tags: spring
categories: [spring,java]
---

使用Spring AOP过程中，可能遇到过下列奇怪问题：
* 类内部方法间调用AOP不起作用
* 类内部方法间调用事务不起作用
* 内部类异步线程调用外层方法事务不起作用

由于Spring的事务底层也是用AOP实现的，因此，这些症状都可以归结为，类内部方法间调用AOP失效。

<!-- more -->

举个官网例子：
```java
public class SimplePojo implements Pojo {

    public void foo() {
        // this next method invocation is a direct call on the 'this' reference
        this.bar();
    }

    public void bar() {
        // some logic...
    }
}

public class Main {

    public static void main(String[] args) {

        ProxyFactory factory = new ProxyFactory(new SimplePojo());
        factory.addInterface(Pojo.class);
        factory.addAdvice(new RetryAdvice());

        Pojo pojo = (Pojo) factory.getProxy();

        // this is a method call on the proxy!
        pojo.foo();
    }
}
```

> The key thing to understand here is that the client code inside the `main(..)` method of the `Main` class has a reference to the proxy. This means that method calls on that object reference are calls on the proxy. As a result, the proxy can delegate to all of the interceptors (advice) that are relevant to that particular method call. However, once the call has finally reached the target object (the `SimplePojo`, reference in this case), any method calls that it may make on itself, such as `this.bar()` or `this.foo()`, are going to be invoked against the `this` reference, and not the proxy. This has important implications. It means that self-invocation is not going to result in the advice associated with a method invocation getting a chance to execute.

在`main`方法中，我们为`SimplePojo`创建了一个代理对象，`pojo`是代理对象的引用。
当调用`foo()`方法时，其实是调用代理对象的方法，我们可以在代理对象中添加拦截器对方法进行增强。
注意，`foo()`方法内部调用`this.bar()`时，是在`this`引用上的，并非代理对象引用，因此，不会被拦截到，无法增强。

那怎么解决这个问题呢？
> The best approach is to refactor your code such that the self-invocation does not happen.

官网推荐做法：干掉内部调用。。。

如果干不掉呢？

那就得跟Spring AOP强绑定起来：
```java
public class SimplePojo implements Pojo {

    public void foo() {
        // this works, but... gah!
        ((Pojo) AopContext.currentProxy()).bar();
    }

    public void bar() {
        // some logic...
    }
}
```

或者在一个bean里面，从Spring容器中手动获取bean进行调用：
```java
@Service("simplePojo")
public class SimplePojo implements Pojo {

    public void foo() {
        // this works, but... gah!
        getService().bar();
    }

    public void bar() {
        // some logic...
    }
}
public PKService getService() {
    return (SimplePojo) SpringContextUtil.getBean("simplePojo");
}
```

还有一种方法，是使用AspectJ的AOP代替Spring AOP的代理方式。在编译器进行代码织入。但是自己用的比较少，这里不做介绍了。

参考：
[5.8.1. Understanding AOP Proxies](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-understanding-aop-proxies)
