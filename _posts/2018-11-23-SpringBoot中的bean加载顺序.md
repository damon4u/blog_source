---
title: SpringBoot中的bean加载顺序
date: 2018-11-23 16:41:50
tags: spring
categories: spring
---

最近在做传统Spring项目到SpringBoot项目迁移过程中，遇到了一些bean加载顺序的问题：
比如一个config中的bean依赖于另一个config中的bean进行初始化，于是查了一些资料，出现了一些新的概念：
* `@Order`
* `@AutoConfigureAfter`
* `@DependsOn`


<!-- more -->

### `@Order`注解
> Before Spring 4.0, the `@Order` annotation was used only for the `AspectJ` execution order. It means the highest order advice will run first. <br>
> Since Spring 4.0, it supports __the ordering of injected components to a collection__. As a result, Spring will inject the auto-wired beans of the same type based on their order value.

在Spring 4.0版本之前，`@Order`注解只能控制AOP的执行顺序，在Spring 4.0之后，它还可以控制集合注入中bean的顺序。
控制AOP顺序很好理解，例如可以在`@Aspect`注解的切面上加入`@Order`注解，控制切面的执行顺序。
还有`@EnableTransactionManagement(order = 10)`，这种写法，由于Spring的事务也是用AOP实现，也可以控制优先级。

下面举个例子说明控制集合注入中bean的顺序。

```java
public interface Rating {
    int getRating();
}

@Component
@Order(1)
public class Excellent implements Rating {

    @Override
    public int getRating() {
        return 1;
    }
}

@Component
@Order(2)
public class Good implements Rating {

    @Override
    public int getRating() {
        return 2;
    }
}

@Component
@Order(Ordered.LOWEST_PRECEDENCE)
public class Average implements Rating {

    @Override
    public int getRating() {
        return 3;
    }
}
```
最后是测试类：
```java
public class RatingRetrieverUnitTest {

    @Autowired
    private List<Rating> ratings;

    @Test
    public void givenOrder_whenInjected_thenByOrderValue() {
        assertThat(ratings.get(0).getRating(), is(equalTo(1)));
        assertThat(ratings.get(1).getRating(), is(equalTo(2)));
        assertThat(ratings.get(2).getRating(), is(equalTo(3)));
    }
}
```
如果不使用`@Order`注解，那ratings集合可能是乱序的。

有一种错误的用法：
先定义两个service：
```java
@Service
public class OrderService1 {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService1.class);

    public OrderService1() {
        LOGGER.info("OrderService1 constructor");
    }

    public String name() {
        return "orderService1";
    }
}

@Service
public class OrderService2 {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService2.class);

    public OrderService2() {
        LOGGER.info("OrderService2 constructor");
    }

    public String name() {
        return "orderService2";
    }
}
```
然后写两个config注入bean：
```java
@Configuration
@Order(1)
public class OrderService1Config {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService1Config.class);

    @Bean
    public OrderService1 orderService1() {
        LOGGER.info("orderService1 init");
        return new OrderService1();
    }
}
@Configuration
@Order(0)
public class OrderService2Config {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService2Config.class);

    @Bean
    public OrderService2 orderService2() {
        LOGGER.info("orderService2 init");
        return new OrderService2();
    }
}
```
本意是想通过`@Order`控制bean的注入顺序，先注入orderService2，再注入orderService1。但是并没有效果。
所以，`@Order`注解放到`@Configuration`中是无法控制bean的注入顺序的。

### `@AutoConfigureAfter`注解
>  Hint for that an `EnableAutoConfiguration` auto-configuration should be applied after other specified auto-configuration classes.

类似的注解还有：
* `@AutoConfigureBefore`
* `@AutoConfigureOrder`

这三个注解是特地用于autoconfigure类的，不能用于普通的配置类。

有必要先说明一下autoconfigure类项目。

通常我们会在主类入口上标注`@SpringBootApplication`注解，或者直接标注`@EnableAutoConfiguration`注解。
这个注解是用来根据类路径中的依赖包猜测需要注入的bean，实现自动注入：
> Enable auto-configuration of the Spring Application Context, attempting to guess and configure beans that you are likely to need. Auto-configuration classes are usually applied based on your classpath and what beans you have defined.


可以理解为，`@EnableAutoConfiguration`是服务于自动注入的bean的，即`spring-boot-starter`中bean的自动加载顺序。
被排序的这些类，都是通过`xxx-spring-boot-autoconfigure`项目中的`src/resources/META-INF/spring.factories`配置文件获取的，这个文件中的配置内容一般为：
```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.github.pagehelper.autoconfigure.PageHelperAutoConfiguration
```
Spring Boot 只会对从这个文件读取的配置类进行排序。

但是不要以为将自己的配置类也配置在`spring.factories`中就能实现排序，如果你的类被自己`Spring Boot`启动类扫描到了，这个类的顺序会优先于所有通过`spring.factories`读取的配置类。
> Auto-configuration is always applied after user-defined beans have been registered.


### `@DependsOn`注解
> Beans on which the current bean depends. Any beans specified are guaranteed to be created by the container before this bean.  <br>
Used infrequently in cases where a bean does not explicitly depend on another through properties or constructor arguments, but rather depends on the side effects of another bean's initialization.<br>
May be used on any class directly or indirectly annotated with org.springframework.stereotype.Component or on methods annotated with Bean.<br>
Using DependsOn at the class level has no effect unless component-scanning is being used. If a DependsOn-annotated class is declared via XML, DependsOn annotation metadata is ignored, and <bean depends-on="..."/> is respected instead.

从java doc中可以看出，`@DependsOn`注解可以用来控制bean的创建顺序，该注解用于声明当前bean依赖于另外一个bean。所依赖的bean会被容器确保在当前bean实例化之前被实例化。
__一般用在一个bean没有通过属性或者构造函数参数显式依赖另外一个bean，但实际上会使用到那个bean或者那个bean产生的某些结果的情况。__

用法
* 直接或者间接标注在带有`@Component`注解的类上面;
* 直接或者间接标注在带有`@Bean`注解的方法上面;
* 使用`@DependsOn`注解到类层面仅仅在使用 __component-scanning__ 方式时才有效;如果带有`@DependsOn`注解的类通过XML方式使用，该注解会被忽略，`<bean depends-on="..."/>`这种方式会生效。

例如，我们有一个`FileProcessor`依赖于`FileReader`和`FileWriter`，`FileReader`和`FileWriter`需要在`FileProcessor`之前初始化：
```java
@Configuration
@ComponentScan("com.baeldung.dependson")
public class Config {

    @Bean
    @DependsOn({"fileReader","fileWriter"})
    public FileProcessor fileProcessor(){
        return new FileProcessor();
    }

    @Bean("fileReader")
    public FileReader fileReader() {
        return new FileReader();
    }

    @Bean("fileWriter")
    public FileWriter fileWriter() {
        return new FileWriter();
    }   
}
```
也可以在`Component`上标注：
```java
@Component
@DependsOn({"filereader", "fileWriter"})
public class FileProcessor {}
```

### 属性注入和构造器注入
上面说到`@DependsOn`注解时提到，它一般用在一个bean没有通过属性或者构造函数参数显式依赖另外一个bean，但实际上会使用到那个bean或者那个bean产生的某些结果的情况。
如果bean直接依赖于另一个bean，我们可以将其通过属性或者构造函数引入进来。
而使用构造函数的方法显示依赖一个bean，能够保证被依赖的bean先初始化。但是属性注入不可以。
> constructor-injection automatically enforces the order and completeness of the instantiated.

因此，我们可以在Component中使用构造函数显示注入依赖的bean：
```java
@Autowired
public MyComponent(@Qualifier("jedisTemplateNew") JedisTemplate jedisTemplateNew) {

}
```
注意，需要使用`@Qualifier`限定bean名称时，不能标注在构造方法上，而是应该标注在参数上。原因跟`@Resource`不能标注构造方法一样，它不知道你要限定哪个参数。

假设有两个service，`OrderService1`依赖于`OrderService2`:
```java
public class OrderService1 {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService1.class);

    private OrderService2 orderService2;

    public OrderService1(OrderService2 orderService2) {
        LOGGER.info("OrderService1 constructor");
        this.orderService2 = orderService2;
    }

    public String name() {
        String name = orderService2.name();
        LOGGER.info("OrderService1 print orderService2 name={}", name);
        return "orderService1";
    }
}

public class OrderService2 {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService2.class);

    public OrderService2() {
        LOGGER.info("OrderService2 constructor");
    }

    public String name() {
        return "orderService2";
    }
}
```
对应的Configuration：
```java
@Configuration
public class OrderService1Config {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService1Config.class);

    @Autowired
    private OrderService2Config orderService2Config;

    @Bean
    public OrderService1 orderService1() {
        LOGGER.info("orderService1 init");
        return new OrderService1(orderService2Config.orderService2());
    }
}

@Configuration
public class OrderService2Config {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService2Config.class);

    @Bean
    public OrderService2 orderService2() {
        LOGGER.info("orderService2 init");
        return new OrderService2();
    }
}
```
输出：
```
c.m.s.config.OrderService1Config         : orderService1 init
c.m.s.config.OrderService2Config         : orderService2 init
c.m.s.service.OrderService2              : OrderService2 constructor
c.m.s.service.OrderService1              : OrderService1 constructor
```
可以看出，`OrderService2`先初始化。

换一种`OrderService1`写法，使用属性注入：
```java
public class OrderService1 {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService1.class);

    private OrderService2 orderService2;

    public void setOrderService2(OrderService2 orderService2) {
        this.orderService2 = orderService2;
    }

    public OrderService1() {
        LOGGER.info("OrderService1 constructor");
    }

    public String name() {
        String name = orderService2.name();
        LOGGER.info("OrderService1 print orderService2 name={}", name);
        return "orderService1";
    }
}

@Configuration
public class OrderService1Config {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService1Config.class);

    @Autowired
    private OrderService2Config orderService2Config;

    @Bean
    public OrderService1 orderService1() {
        LOGGER.info("orderService1 init");
        OrderService1 orderService1 = new OrderService1();
        orderService1.setOrderService2(orderService2Config.orderService2());
        return orderService1;
    }
}
```
输出：
```
c.m.s.config.OrderService1Config         : orderService1 init
c.m.s.service.OrderService1              : OrderService1 constructor
c.m.s.config.OrderService2Config         : orderService2 init
c.m.s.service.OrderService2              : OrderService2 constructor
```
可以看出，`OrderService2`并没有先初始化。

当然，`OrderService1Config`也可以使用构造器注入：
```java
@Configuration
public class OrderService1Config {
    private static final Logger LOGGER = LoggerFactory.getLogger(OrderService1Config.class);

    private final OrderService2Config orderService2Config;

    @Autowired
    public OrderService1Config(OrderService2Config orderService2Config) {
        this.orderService2Config = orderService2Config;
    }

    @Bean
    public OrderService1 orderService1() {
        LOGGER.info("orderService1 init");
        OrderService1 orderService1 = new OrderService1();
        orderService1.setOrderService2(orderService2Config.orderService2());
        return orderService1;
    }
}
```

参考：
[@Order in Spring](https://www.baeldung.com/spring-order)
[Controlling Bean Creation Order with @DependsOn Annotation](https://www.baeldung.com/spring-depends-on)
