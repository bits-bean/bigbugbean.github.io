---
title: Spring IOC容器和Bean的拓展点
tags: Spring Framework
typora-copy-images-to: ..\..\assets\blog_img
typora-root-url: ..\..
---

## 概述

Spring Framework提供了多个IOC容器和Bean的拓展点，包括在容器启动和停止、bean的初始化和销毁等，方便开发者去实现一些自定义的逻辑。但Spring的官方文档对于这些拓展点的说明比较零散，没有把所有的拓展点串起来，这篇文章的目的就是尽可能地列出Spring Framework的所有拓展点，并分析它们的执行顺序和应用场景。

## Bean的拓展点

### Bean生命周期回调

#### 初始化回调

bean可以在容器设置了其所有必要属性后执行初始化工作，且有两种实现方式：

- 实现`InitializingBean`接口

  `InitializingBean`接口指定了一个方法：
  
  ```java
  void afterPropertiesSet() throws Exception;
  ```
  
  此方式不推荐，因为会与Spring框架耦合。

-  使用`@PostConstruct`注解

  推荐使用此方式，因为`@PostConstruct`注解是[JSR-250](https://en.wikipedia.org/wiki/JSR_250)规范定义的注解，不会与具体框架耦合

#### 销毁回调

bean可以让一个在包含它的容器被销毁时获得一个回调，且有两种实现方式：

- 实现`DisposableBean`接口

  `DisposableBean`接口指定了一个方法：

  ```java
  void destroy() throws Exception;
  ```

  此方式不推荐，因为会与Spring框架耦合。

- 使用`@PreDestroy`注解

  推荐使用此方式，因为`@PreDestroy`注解是[JSR-250](https://en.wikipedia.org/wiki/JSR_250)规范定义的注解，不会与具体框架耦合

#### 自定义初始化/销毁方法

除了上述的初始化回调和销毁回调，还可以自定义bean的初始化/销毁方法，而且此机制也是解耦的。这两种机制效果是差不多的，但是初始化回调和销毁回调会在自定义的初始化/销毁方法之前执行。

下面代码展示了基于Java配置的自定义初始化/销毁方法：

```java
public class BeanOne {

    public void init() {
        // initialization logic
    }
}

public class BeanTwo {

    public void cleanup() {
        // destruction logic
    }
}

@Configuration
public class AppConfig {

    @Bean(initMethod = "init")
    public BeanOne beanOne() {
        return new BeanOne();
    }

    @Bean(destroyMethod = "cleanup")
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```

## IOC容器的拓展点

### 后置处理器

#### BeanPostProcessor

`BeanPostProcessor`顾名思义是bean的后置处理器，可以理解为在bean实例化后做一些处理工作。你只要实现`BeanPostProcessor`接口并自定义一些处理逻辑，可以是改变bean的属性、用代理包装原来的bean等，这些逻辑就会作用在Spring的所有bean实例上，当然除了`BeanPostProcessor`自身的实例（`BeanPostProcessor`的实例也会被Spring当作bean一样管理，但与普通的bean不同，Spring容器在启动时就会实例化BeanPostProcessor-Bean和它们直接引用的bean）。

`BeanPostProcessor`接口定义了两个回调方法`postProcessBeforeInitialization`和`postProcessAfterInitialization`，这两个方法分别是在bean初始化之前和之后回调的。在bean初始化时，容器回调用`InitializingBean`的`afterPropertiesSet`方法和自定义的`init-method`，也就是说，这几个方法间的调用顺序是`postProcessBeforeInitialization`->`afterPropertiesSet`->`init-method`->`postProcessAfterInitialization`。

`AutowiredAnnotationBeanPostProcessor`就是`BeanPostProcessor`的应用之一：`AutowiredAnnotationBeanPostProcessor`是`BeanPostProcessor`实现类，它的作用替带有`@Autowired`等注解的字段或setter方法注入相应的依赖。

更多细节和示例可以参阅[Customizing Beans by Using a `BeanPostProcessor`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-extension-bpp)

#### BeanFactoryPostProcessor

`BeanFactoryPostProcessor`和`BeanPostProcessor`的语义很相似，但主要的区别是：`BeanPostProcessor`是作用于实例化后的bean，而`BeanFactoryPostProcessor`是作用于bean的配置元数据。也就是说，Spring IoC容器允许`BeanFactoryPostProcessor`读取配置元数据，并可能在容器实例化除BeanFactoryPostProcessor实例之外的任何bean之前更改配置元数据。

`PropertySourcesPlaceholderConfigurer`就是通过实现`BeanFactoryPostProcessor`来替换bean配置元数据中的占位符，来达到配置和程序代码分离的目的。

更多细节和示例可以参阅[Customizing Configuration Metadata with a `BeanFactoryPostProcessor`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-extension-factory-postprocessors)

### 容器生命周期



### 事件机制

下表列举了Spring提供的标准事件，这些事件都是实现了`ApplicationEvet`接口的，bean只要实现`ApplicationListener`接口就会在事件触发时收到通知。

| 标准事件                     | 描述                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| `ContextStartedEvent`        | 在ApplicationContext启动时触发。启动时具体指的是所有实现了Lifecycle接口的bean接收到明确的启动信号，通常这个信号是用来重启那些被停止的bean，但还可用于启动尚未配置为自动启动的组件，如初始化时尚未启动的组件。 |
| `ContextRefreshedEvent`      | 在ApplicationContext初始化完成或刷新完成时触发。初始化完成具体是指所有bean已经加载，post-processor bean已激活，singleton bean已完成预实例化并且ApplicationContext已经准备就绪了。只要容器没有关闭，就可以多次触发刷新，只要具体容器实现类支持“热”刷新。 |
| `ContextStoppedEvent`        | 在ApplicationContext停止时触发。停止意味着所有实现了Lifecycle接口的bean接收到明确的停止信号，已停止的容器也可能通过调用start()方法重新启动。 |
| `ContextClosedEvent`         | 在ApplicationContext关闭时或者通过JVM的shutdown hook触发。关闭意味着所有singleton bean都被销毁，容器一旦关闭，表示到达了生命周期的最后环节，不能再被刷新或启动。 |
| `RequestHandledEvent`        | 在HTTP请求完成后触发。此事件是一个特定于 Web 的事件，旨在告诉所有 bean 一个 HTTP 请求已被处理，仅仅适用于使用DispatcherServlet的web应用。 |
| `ServletRequestHandledEvent` | RequestHandledEvent的子类，增加了一些servlet特定的上下文信息。 |



## 拓展点的执行顺序


> 参考文献
>
> [Customizing Beans by Using a `BeanPostProcessor`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-extension-bpp)
>
> [Customizing Configuration Metadata with a `BeanFactoryPostProcessor`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-extension-factory-postprocessors)