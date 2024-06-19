---
layout: post
title: "zfoo库 event"
subtitle: "zfoo-event 源码学习"
date: 2024-06-16 15:00:00
author: "Momoka7"
header-style: text
catalog: true
tags:
  - zfoo
  - event
  - java
---

# 基础使用

## 在监听类中注册事件

一些使用规则：

1. 使用`@EventReceiver`注解标注监听的方法，可以指定事件的处理总线类型；

2. 方法参数必须是实现了`IEvent`接口的类，通过参数类型来区分所监听的事件；

3. 方法名称必须是`on`开头；

```java
    @Component
public class MyController1 {
    private static final Logger logger = LoggerFactory.getLogger(MyController1.class);

    // 事件会被当前线程立刻执行，注意日志打印的线程号
    @EventReceiver
    public void onMyNoticeEvent(MyNoticeEvent event) {
        logger.info("方法1同步执行事件：" + event.getMessage());
    }

    @EventReceiver(Bus.AsyncThread)
    public void onAnotherEvent(AnotherEvent event) {
        logger.info("方法2异步执行事件：" + event.getMessage());
    }

}
```

## 触发事件

```java
    // 使用EventBus的静态方法post触发事件
    EventBus.post(MyNoticeEvent.valueOf("我的事件"));
    EventBus.post(AnotherEvent.valueOf("另一个事件"));
```

# 主要类 `EventRegisterProcessor`

## xml 中 bean，及其自定义命名空间解析过程

若在`application.xml`配置文件中定义了`event` bean，且使用了`event`命名空间：

```xml

<beans ...
       xmlns:event="http://www.zfoo.com/schema/event"

       xsi:schemaLocation="
    ...
    http://www.zfoo.com/schema/event
    http://www.zfoo.com/schema/event-1.0.xsd">
    <context:component-scan base-package="com.zfoo.event"/>
    <event:event id="eventBus"/>
</beans>
```

### 解析命名空间

所需组件

![截图](/img/in-post/zfoo-event/f56813c17285ebab4284007c2e08ed34.png)

#### 流程概述

1. 加载 XML 配置文件：Spring 应用上下文加载 XML 配置文件。
2. 解析 XML 元素：Spring 解析器解析 XML 文件内容。
3. 遇到自定义命名空间：当遇到自定义命名空间前缀的元素时，解析器根据 `META-INF/spring.handlers` 找到对应的 `NamespaceHandler`。`META-INF/spring.schemas`注册 XSD 文件。
4. 创建和初始化 NamespaceHandler：使用反射机制创建 NamespaceHandler 实例，并调用其 init() 方法。
5. 解析自定义标签：使用 NamespaceHandler 注册的 BeanDefinitionParser 解析自定义标签，并生成 BeanDefinition。
6. 注册 BeanDefinition：将生成的 BeanDefinition 注册到 Spring 容器中。

其中，例如：

**META-INF/spring.handlers**：

```
http\://www.zfoo.com/schema/event=com.zfoo.event.schema.NamespaceHandler
```

**META-INF/spring.schemas**：

```
http\://www.zfoo.com/schema/event-1.0.xsd=event-1.0.xsd
```

**event-1.0.xsd：**

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no" ?>

<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema"

            xmlns="http://www.zfoo.com/schema/event"
            targetNamespace="http://www.zfoo.com/schema/event"

            elementFormDefault="qualified"
            attributeFormDefault="unqualified">

    <xsd:element name="event">
        <xsd:complexType>
            <xsd:attribute name="id" type="xsd:string" use="required"/>
        </xsd:complexType>
    </xsd:element>
</xsd:schema>
```

## EventRegisterProcessor 的解析过程（Receiver 的解析过程）

```java
NamespaceHandler.init()->
  new EventDefinitionParser()->
EventDefinitionParser.parse()
```

在`EventDefinitionParser.parse()`方法中：

注册了`EventContext`和`EventRegisterProcessor`的 bean

```java
@Override
public AbstractBeanDefinition parse(Element element, ParserContext parserContext) {
    Class<?> clazz;
    String name;
    BeanDefinitionBuilder builder;

    // 注册EventContext
    clazz = EventContext.class;
    name = StringUtils.uncapitalize(clazz.getName());
    builder = BeanDefinitionBuilder.rootBeanDefinition(clazz);
    parserContext.getRegistry().registerBeanDefinition(name, builder.getBeanDefinition());

    // 注册EventRegisterProcessor，event事件处理
    clazz = EventRegisterProcessor.class;
    name = StringUtils.uncapitalize(clazz.getName());
    builder = BeanDefinitionBuilder.rootBeanDefinition(clazz);
    parserContext.getRegistry().registerBeanDefinition(name, builder.getBeanDefinition());

    return builder.getBeanDefinition();
}
```

`EventRegisterProcessor.java`：

由于其继承`BeanPostProcessor`并重写了`postProcessAfterInitialization`，故此方法会在在 bean 初始化之后调用，**扫描 bean 中带有 @EventReceiver 注解的方法，检查这些方法的有效性，并将它们注册到事件总线 EventBus 中，以便于处理特定的事件类型。**

```java
public class EventRegisterProcessor implements BeanPostProcessor {

    private static final Logger logger = LoggerFactory.getLogger(EventRegisterProcessor.class);

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        //使用反射获取 bean 的类和所有标注了 @EventReceiver 注解的方法。
        //如果没有标注了 @EventReceiver 注解的方法，则直接返回该 bean。

        //验证类是否为 POJO 类（Plain Old Java Object），并输出警告日志。
        //遍历所有标注了 @EventReceiver 注解的方法，验证每个方法是否符合以下要求：
          //方法必须有且仅有一个参数。
          //方法的参数类型必须实现 IEvent 接口。
          var eventClazz = (Class<? extends IEvent>) paramClazzs[0];
          //方法必须是 public 的。
          //方法不能是 static 的。
          //方法名必须符合约定格式，即 on 加上事件类的简单名称。

          //从注解中获取事件总线标识。
          var bus = method.getDeclaredAnnotation(EventReceiver.class).value();
          //将事件接收器注册到 EventBus 中。
          var receiverDefinition = new EventReceiverDefinition(bean, method, bus, eventClazz);
          //创建并增强事件接收器定义（EventReceiverDefinition）。
          var enhanceReceiverDefinition = EnhanceUtils.createEventReceiver(receiverDefinition);

          // key:class类型 value:观察者 注册Event的receiverMap中
          EventBus.registerEventReceiver(eventClazz, enhanceReceiverDefinition);
        return bean;
    }
}
```

### 注册到事件总线 EventBus

```java
//从注解中获取事件总线标识。
var bus = method.getDeclaredAnnotation(EventReceiver.class).value();
//将事件接收器注册到 EventBus 中。
var receiverDefinition = new EventReceiverDefinition(bean, method, bus, eventClazz);
//创建并增强事件接收器定义（EventReceiverDefinition）。
var enhanceReceiverDefinition = EnhanceUtils.createEventReceiver(receiverDefinition);

// key:class类型 value:观察者 注册Event的receiverMap中
EventBus.registerEventReceiver(eventClazz, enhanceReceiverDefinition);
```

这里的 Bus 是一个枚举类，目前有 2 种实现（同步事件、异步事件），虚拟线程在 java21 实装。

```java
public enum Bus {
    CurrentThread,
    AsyncThread,
    VirtualThread;//in java21 implemented
}
```

`EventReceiverDefinition`是封装了事件监听方法、对应对象、Bus 和`IEvent`监听事件类型（监听方法的参数，通过此类型了区别其感兴趣的事件类型）的一个对象

```java
public EventReceiverDefinition(Object bean, Method method, Bus bus, Class<? extends IEvent> eventClazz) {
  this.bean = bean;
  this.method = method;
  this.bus = bus;
  this.eventClazz = eventClazz;
  ReflectionUtils.makeAccessible(this.method);
}
```

#### 创建并增强事件接收器定义

`EnhanceUtils.createEventReceiver`使用了 `Javassist `库来动态生成一个实现 `IEventReceiver `接口的类。这个动态生成的类会在**运行时调用特定 bean 的方法来处理事件**，而这个生成的类可视作**观察者**。其动态生成的过程如下：

![截图](/img/in-post/zfoo-event/7cb4687e62f2092af0610e4a71ffc035.png)

1. 定义类，并添加 IEventReceiver 接口

`IEventReceiver.java`定义如下，其包含了两个方法，**这两个方法的实现是在后续动态生成的。**

```java
public interface IEventReceiver {
    Bus bus();
    void invoke(IEvent event);
}

```

2. 定义类中的一个成员

这个成员即为监听了事件`@EventReceiver`的 javabean

3. 定义构造器

接受 javabean 初始化

4&5. 定义类实现的接口方法`invoker`和`bus`

从这两个方法的方法体可以看出`invoker`将监听的方法代理给了此动态生成的`IEventReceiver`对象，`bus`方法获取总线类型。

![截图](/img/in-post/zfoo-event/8c9a1108b63a7e0b4ae55895b9ddc2f2.png)

#### 注册 Event 到 receiverMap 中

若`eventType`没有注册过，则先新建一个`ArrayList<List<IEventReceiver>>`，再添加对应的观察者到此`ArrayList`中。

此`receiverMap`用于记录有哪些`IEventReceiver`对哪些`eventType`感兴趣，**在`eventType`触发并要派发事件时，直接遍历`receiverMap`中与`eventType`对应的`IEventReceiver`列表即可。**

```java
EventBus.registerEventReceiver(eventClazz, enhanceReceiverDefinition);
->
public static void registerEventReceiver(Class<? extends IEvent> eventType, IEventReceiver receiver) {
    receiverMap.computeIfAbsent(eventType, it -> new ArrayList<>(1)).add(receiver);
}
```

在`EventBus`中，维护了一个静态常量`receiverMap`

`key`:`eventType`，监听方法参数的 class `value`:观察者，上一步获取的`enhanceReceiverDefinition`

```java
private static final Map<Class<? extends IEvent>, List<IEventReceiver>> receiverMap = new HashMap<>();
```

## 事件触发

使用如下格式来触发事件

```java
EventBus.post(MyNoticeEvent.valueOf("我的事件"));
```

其中`EventBus.post`是一个静态方法，`MyNoticeEvent`是实现了`IEvent`接口的事件类，`valurOf`是其静态方法，返回一个`MyNoticeEvent`类的对象。

### 触发流程（post 方法）

```java
public static void post(IEvent event) {
  if (event == null) {
      return;
  }
  var clazz = event.getClass();
  // 从receiverMap中获取对event感兴趣的方法代理列表List<IEventReceiver>
  var receivers = receiverMap.get(clazz);
  if (CollectionUtils.isEmpty(receivers)) {
      return;
  }
  for (var receiver : receivers) {
      switch (receiver.bus()) {//通过其使用的总线类型来触发
          case CurrentThread://同步触发
              //其实就是调用了receiver.invoke(event)
              //可以理解为将event传入监听事件的方法中，并调用
              doReceiver(receiver, event);
              break;
          case AsyncThread://异步触发
              //在线程池中随机选择一个线程来执行异步任务
              //此线程池由EventBus维护，其大小为可用处理器个数*2
              execute(event.executorHash(), () -> doReceiver(receiver, event));
              break;
          case VirtualThread:
              logger.error("waiting for java 21 virtual thread");
              break;
      }
  }
}
```
