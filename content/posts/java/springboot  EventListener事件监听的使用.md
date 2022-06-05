---

title: SpringBoot 事件监听

date: 2020-01-20 08:49:51

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Spring Boot' ]

categories: [ 'Spring','Spring Boot' ]

---

`EventListener`事件触发和监听器可以对代码解耦，在一些与业务无关的，通用的操作方法，我们可以把它设计成事件监听器，像通知，消息这些模块都可以这样设计。

# Java的事件机制

java中的事件机制一般包括3个部分：`EventObject`，`EventListener`和`Source`。

## EventObject

`java.util.EventObject`是事件状态对象的基类，它封装了事件源对象以及和事件相关的信息。所有java的事件类都需要继承该类。

## EventListener

`java.util.EventListener`是一个标记接口，就是说该接口内是没有任何方法的。所有事件监听器都需要实现该接口。事件监听器注册在事件源上，当事件源的属性或状态改变的时候，调用相应监听器内的回调方法。

## Source

事件源不需要实现或继承任何接口或类，它是事件最初发生的地方。因为事件源需要注册事件监听器，所以事件源内需要有相应的盛放事件监听器的容器。

java的事件机制是一个观察者模式。大家可以根据这个模式，自己实现一个。可以看看这篇博文:《java事件机制》一个很简单的实例。

# Spring的事件

> `ApplicationEvent`以及`Listener`是Spring为我们提供的一个事件监听、订阅的实现，内部实现原理是观察者设计模式，设计初衷也是为了系统业务逻辑之间的解耦，提高可扩展性以及可维护性。[context-functionality-events](https://docs.spring.io/spring/docs/5.2.3.RELEASE/spring-framework-reference/core.html#context-functionality-events)

+ `ApplicationEvent`就是Spring的事件接口

+ `ApplicationListener`就是Spring的事件监听器接口，所有的监听器都实现该接口

+ `ApplicationEventPublisher`是Spring的事件发布接口，ApplicationContext实现了该接口

+ `ApplicationEventMulticaster`就是Spring事件机制中的事件广播器，默认实现SimpleApplicationEventMulticaster

在Spring中通常是`ApplicationContext`本身担任监听器注册表的角色，在其子类`AbstractApplicationContext`中就聚合了事件广播器`ApplicationEventMulticaster`和事件监听器`ApplicationListnener`，并且提供注册监听器的`addApplicationListnener`方法。

其执行的流程大致为：

> 当一个事件源产生事件时，它通过事件发布器`ApplicationEventPublisher`发布事件，然后事件广播器`ApplicationEventMulticaster`会去事件注册表`ApplicationContext`中找到事件监听器`ApplicationListnene`r，并且逐个执行监听器的`onApplicationEvent`方法，从而完成事件监听器的逻辑。

在Spring中，使用注册监听接口，除了继承`ApplicationListener`接口外，还可以使用注解`@EventListener`来监听一个事件，同时该注解还支持SpEL表达式，来触发监听的条件，比如只接受编码为001的事件，从而实现一些个性化操作。下文示例中会简单举例下。

简单来说，在Java中，通过`java.util.EventObject`来描述事件，通过`java.util.EventListener`来描述事件监听器，在众多的框架和组件中，建立一套事件机制通常是基于这两个接口来进行扩展。

# SpringBoot的默认启动事件

+ `ApplicationStartingEvent`：springboot启动开始的时候执行的事件

+ `ApplicationEnvironmentPreparedEvent`：spring boot对应Enviroment已经准备完毕，但此时上下文context还没有创建。在该监听中获取到ConfigurableEnvironment后可以对配置信息做操作，例如：修改默认的配置信息，增加额外的配置信息等等。

+ `ApplicationPreparedEvent`：spring boot上下文context创建完成，但此时spring中的bean是没有完全加载完成的。在获取完上下文后，可以将上下文传递出去做一些额外的操作。值得注意的是：**在该监听器中是无法获取自定义bean并进行操作的**。

+ `ApplicationReadyEvent`：springboot加载完成时候执行的事件。、

+ `ApplicationFailedEvent`：spring boot启动异常时执行事件。

# 自定义事件发布和监听

## MAVEN引入

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

## 定义事件源

```java
import org.springframework.context.ApplicationEvent;

/**
 * <p>
 * 自定义事件
 * </P>
 *
 * @author bing_huang
 * @since V1.0
 */
public class DemoEvent extends ApplicationEvent {
    private String message;

    public DemoEvent(Object source, String message) {
        super(source);
        this.message = message;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

## 事件处理程序（监听类）

1. 实现ApplicationListnener方式
   
   ```java
   import org.springframework.context.ApplicationListener;
   import org.springframework.stereotype.Component;
   
   /**
    * <p>
    * 事件监听器
    * </P>
    *
    * @author bing_huang
    * @since V1.0
    */
   @Component
   public class DemoListener implements ApplicationListener<DemoEvent> {
   
      @Override
      public void onApplicationEvent(DemoEvent demoEvent) {
           String msg = demoEvent.getMessage();
           System.out.println("(bean-demoListener implements ApplicationListener<DemoEvent>)接收到了bean-demoPublisher发布的消息:" + msg);
       }
   }
   ```

2. @EventListener注解方式。
   
   ```java
   import org.springframework.context.event.EventListener;
   import org.springframework.scheduling.annotation.Async;
   import org.springframework.stereotype.Component;
   
   /**
    * <p>
    *     事件监听器
    * </P>
    *
    * @author bing_huang
    * @since V1.0
    */
   @Component
   public class DemoListener2 {
   ```

       @EventListener
       @Async
       public void onApplicationEvent(DemoEvent demoEvent) {
           String msg = demoEvent.getMessage();
           System.out.println("(bean-demoListener  @EventListener)接收到了bean-demoPublisher发布的消息:" + msg);
       }

   }

```
**注意:不管是实现ApplicationListener或者注解方式都需要交由spring管理`@Component`**

## 事件触发

 1. 通过`ApplicationContext`

 事件发布是由ApplicationContext对象管控的，在事件发布之前需要注入 ApplicationContext对象，然后通过 publishEvent 方法完成事件发布。 
 ```java
 import org.springframework.beans.factory.annotation.Autowired;
 import org.springframework.context.ApplicationContext;
 import org.springframework.stereotype.Component;

 /**
  * <p>
  * 消息发布者
  * </P>
  *
  * @author bing_huang
  * @since V1.0
  */
 @Component
 public class DemoPublisher {

     private final ApplicationContext applicationContext;

     @Autowired
     public DemoPublisher(ApplicationContext applicationContext) {
         this.applicationContext = applicationContext;
     }

     public void publish(String message) {
         applicationContext.publishEvent(new DemoEvent(this, message));
     }

 }
 ```

2. 通过`ApplicationEventPublisher`方式发布

```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class DemoPublisherTest {

    @Autowired
    DemoPublisher demoPublisher;
    @Autowired
    ApplicationEventPublisher publisher;

    @Test
    public void publish() {
        demoPublisher.publish("test");

    }

    @Test
    public void publish2() {
        publisher.publishEvent(new DemoEvent(this, "hello"));
    }
}
```

# 测试

```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class DemoPublisherTest {

    @Autowired
    DemoPublisher demoPublisher;
    @Autowired
    ApplicationEventPublisher publisher;

    @Test
    public void publish() {
        demoPublisher.publish("test");

    }

    @Test
    public void publish2() {
        publisher.publishEvent(new DemoEvent(this, "hello"));
    }
}
```

**[transaction-event](https://docs.spring.io/spring/docs/5.2.3.RELEASE/spring-framework-reference/data-access.html#transaction-event)** 从Spring 4.2开始，事件的侦听器可以绑定到事务的某个阶段。典型的示例是在事务成功完成后处理事件
