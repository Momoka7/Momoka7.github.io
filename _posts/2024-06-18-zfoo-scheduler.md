---
layout: post
title: "zfoo库 scheduler"
subtitle: "zfoo库 scheduler源码学习"
date: 2024-06-18 22:40:00
author: "Momoka7"
header-style: text
catalog: true
tags:
  - zfoo
  - scheduler
  - java
  - cron
---

# 基础使用

在需要定时调度的方法上添加注解`@Scheduler`，定义其`cron`表达式属性（字符串）即可。

```java
@Scheduler(cron = "0/5 * * * * ?")
public void cronScheduler1() {
    logger.info("scheduler1 每5秒时间调度任务");
}

@Scheduler(cron = "0,10,20,40 * * * * ?")
public void cronScheduler2() {
    logger.info("scheduler2 每分钟的10秒，20秒，40秒调度任务");
}
```

# cron

Cron 表达式用于调度定时任务，常见于类 Unix 操作系统（如 Linux）和一些定时任务调度系统中。Cron 表达式由**五个或六个字段**组成，用于定义任务的执行时间。以下是标准的五个字段格式：

```
* * * * *
- - - - -
| | | | |
| | | | +---- 星期几 (0 - 7) (星期天为0或7)
| | | +------ 月份 (1 - 12)
| | +-------- 日期 (1 - 31)
| +---------- 小时 (0 - 23)
+------------ 分钟 (0 - 59)
```

如果包含六个字段，通常是指年份字段，格式如下：

```
* * * * * *
- - - - - -
| | | | | |
| | | | | +---- 年 (1970 - 2099)
| | | | +------ 星期几 (0 - 7) (星期天为0或7)
| | | +-------- 月份 (1 - 12)
| | +---------- 日期 (1 - 31)
| +------------ 小时 (0 - 23)
+-------------- 分钟 (0 - 59)
```

每个字段可以包含以下类型的值：

1. **整数**：表示特定的时间点。例如，`5` 在分钟字段中表示每小时的第 5 分钟。
2. **范围**：用连字符`-`表示范围。例如，`1-5` 表示从 1 到 5。
3. **列表**：用逗号`,`分隔多个值。例如，`1,3,5`表示第 1、第 3 和第 5。
4. 通配符`*` ：表示每一个可能的值。例如，`*`在小时字段中表示每小时。
5. **步进值**：用斜杠`/`表示步进。例如，`*/5`表示每 5 个单位。
6. **特殊字符**：
   - `L`：在日期字段表示每月的最后一天，在星期字段表示每周的最后一天（星期六）。
   - `W`：在日期字段表示最近的工作日。
   - `#`：在星期字段表示第几个星期几。例如，`3#2`表示每个月的第二个星期三。
   - `?`：只在日期和星期字段中使用，表示不指定值，用于避免冲突。

下面是一些示例及其含义：

1. `0 0 * * *`：每天午夜 0 点执行任务。
2. `0 12 * * *`：每天中午 12 点执行任务。
3. `0 0 1 * *`：每月的 1 日午夜 0 点执行任务。
4. `0 0 * * 0`：每周日午夜 0 点执行任务。
5. `*/15 * * * *`：每 15 分钟执行一次任务。
6. `0 9-17 * * 1-5`：每个工作日的 9 点到 17 点（每小时的 0 分）执行任务。
7. `0 0 1 1 *`：每年的 1 月 1 日午夜 0 点执行任务。

# 源码分析

和`event`模块类似，使用`xml`引入`scheduler`：

```xml
<scheduler:scheduler id="schedulerManager"/>
```

其也自定义了命名空间，并在解析命名空间时添加了核心的 bean：`SchedulerContext`

## SchedulerContext

`SchedulerContext`类管理调度任务的注册和生命周期，通过监听`Spring`上下文事件，在上下文启动时初始化并注入调度任务，在上下文关闭时优雅地停止调度任务。通过反射和注解处理，灵活地管理调度任务的方法。

首先看看其类的继承关系，其实现了两个接口：

```java
public class SchedulerContext implements ApplicationListener<ApplicationContextEvent>, Ordered
```

`ApplicationListener<ApplicationContextEvent>`：监听 Spring 应用上下文事件。
`Ordered`：定义 Spring Bean 加载顺序，确保当前类在其他 Bean 之后加载。

成员变量：

```java
private static final Logger logger = LoggerFactory.getLogger(SchedulerContext.class);

//单例模式
private static SchedulerContext instance;

private static boolean stop = false;

//Spring上下文
private ApplicationContext applicationContext;
```

### shutdown

停止调度器，使用反射获取并关闭`SchedulerBus`中的`executor`线程池。

```java
public synchronized static void shutdown() {
    if (stop) {
        return;
    }
    stop = true;

    try {
        // 获取SchedulerBus中的executor，此executor是一个静态常量
        // executor默认是只有一个单线程的线程池
        Field field = SchedulerBus.class.getDeclaredField("executor");
        ReflectionUtils.makeAccessible(field);
        var executor = (ScheduledExecutorService) ReflectionUtils.getField(field, null);
        ThreadUtils.shutdown(executor);
    } catch (Throwable e) {
        logger.error("Scheduler thread pool failed shutdown.", e);
        return;
    }

    logger.info("Scheduler shutdown gracefully.");
}
```

### onApplicationEvent

监听`Spring`的事件

```java
@Override
public void onApplicationEvent(ApplicationContextEvent event) {
    if (event instanceof ContextRefreshedEvent) {
        // 初始化上下文
        SchedulerContext.instance = this;
        instance.applicationContext = event.getApplicationContext();
        inject();
        // 启动位于SchedulerBus的static静态代码块中
    } else if (event instanceof ContextClosedEvent) {
        // 关闭事件，停止调度器
        shutdown();
    }
}
```

### inject

注入调度任务

```java
public void inject() {
  //扫描所有@Component注解的Bean，
    //查找带@Scheduler注解的方法并进行注册。
    //校验方法是否符合要求：无参数、public、非static、方法名以cron开头。
    //创建SchedulerDefinition实例并注册到SchedulerBus(添加到其CopyOnWriteArrayList中)
}
```

### getOrder

返回最低优先级，确保`SchedulerContext`最后加载（**需求保证其他 Bean 都注册完毕后，才能扫描全部需要调度的 bean**）。

## SchedulerBus

都是静态成员和方法，负责管理和执行调度任务。它通过一个单线程的调度器执行器（`ScheduledExecutorService`）来**定期检查和触发**注册的调度任务。

### 类和成员变量定义

```java
private static final Logger logger = LoggerFactory.getLogger(SchedulerBus.class);
//保存所有注册的调度任务(方法)定义。
private static final List<SchedulerDefinition> schedulerDefList = new CopyOnWriteArrayList<>();
//单线程调度执行器，用于定期执行调度任务。
private static final ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor(new SchedulerThreadFactory(1));
// 记录调度执行器的线程ID。
private static long threadId = 0;
// 上一次触发调度任务的时间戳。
private static long lastTriggerTimestamp = 0L;
// 调度任务列表中最小的触发时间戳。
private static long minTriggerTimestamp = 0L;
```

### 静态初始化块

```java
//启动一个定时任务，每秒调用一次triggerPerSecond方法，
//用于检查和触发调度任务。
static {
    executor.scheduleAtFixedRate(() -> {
        try {
            triggerPerSecond();
        } catch (Exception e) {
            logger.error("scheduler triggers an error.", e);
        }
    }, 0, TimeUtils.MILLIS_PER_SECOND, TimeUnit.MILLISECONDS);
}
```

### SchedulerThreadFactory

静态内部类，自定义线程工厂，用于创建调度线程并设置线程属性，如线程名、优先级和异常处理器。

#### refreshMinTriggerTimestamp

刷新调度任务列表中最小的触发时间戳。**会在有新的调度任务注册时调用。**

```java
public static void refreshMinTriggerTimestamp() {
    var minTimestamp = Long.MAX_VALUE;
    for (var scheduler : schedulerDefList) {
        if (scheduler.getTriggerTimestamp() < minTimestamp) {
            minTimestamp = scheduler.getTriggerTimestamp();
        }
    }
    minTriggerTimestamp = minTimestamp;
}
```

#### triggerPerSecond

```java
//查看调度任务列表schedulerDefList，若为空则直接返回
//系统时间被回调：
  //重新计算每个调度任务的下次触发时间。
  //调用 refreshMinTriggerTimestamp 刷新全局最小触发时间戳。
//更新上次触发时间戳为当前时间（系统时间向后调整也一定会触发）
//如果当前时间小于最小触发时间戳，说明没有任务需要在当前时间触发，直接返回。
//遍历调度任务列表
  //获取每个任务的触发时间戳。
  //如果触发时间戳小于或等于当前时间，说明该任务需要触发
    //调用任务的调度方法 invoke() 触发任务
    //重新计算任务的下一次触发时间，并更新任务的触发时间戳。
  //更新 minTimestamp 为所有任务中的最小触发时间戳。
```

#### 其他方法

都是可以单独调用的静态方法

```java
// 不断执行的周期循环任务
public static ScheduledFuture<?> scheduleAtFixedRate(Runnable runnable, long period, TimeUnit unit) {
    return executor.scheduleAtFixedRate(SafeRunnable.valueOf(runnable), 0, period, unit);
}
//固定延迟执行的任务
public static ScheduledFuture<?> schedule(Runnable runnable, long delay, TimeUnit unit) {
    return executor.schedule(SafeRunnable.valueOf(runnable), delay, unit);
}

//cron表达式执行的任务
public static void scheduleCron(Runnable runnable, String cron) {
    if (SchedulerContext.isStop()) {
        return;
    }
    registerScheduler(SchedulerDefinition.valueOf(cron, runnable));
}

//获取当前线程的executor
public static Executor threadExecutor(long currentThreadId) {
    return threadId == currentThreadId ? executor : null;
}
```

## SchedulerDefinition

调度任务的载体，一般在`SchedulerContext`的解析过程中创建，通过`SchedulerDefinition.valueOf(String cron, Object bean, Method method)`方法构造。

## TimeUtils

负责时间的计算、校验等，在其`static`块中调用了`SchedulerBus.refreshMinTriggerTimestamp()`，以使`SchedulerBus`类被加载

# 其他

类的加载顺序：`SchedulerDefinition->TimeUtils->SchedulerBus`

在`SchedulerContext`->`SchedulerDefinition.valueOf()`->`TimeUtils.nextTimestampByCronExpression`->`TimeUtils.static{SchedulerBus.refreshMinTriggerTimestamp()}`->`SchedulerBus.static{//启动任务调度周期检查线程}`

从这个过程中可以看出，只有当有`SchedulerDefinition`创建，即 cron 可调度任务创建时才会启用`SchedulerBus`
