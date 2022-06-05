---

title: SpringBoot 定时任务动态配置

date: 2019-12-15 11:24:39

author: hb0730

authorLink: https://blog.hb0730.com

tags: ['Spring','Spring Task']

categories: ['Spring','Spring Boot']

---

# 前言

 由于上周工作的需求需要使用springboot的`@Scheduled`做成可配置的，动态的，有幸看到[胡峻峥](https://www.cnblogs.com/hujunzheng/p/10353390.html#autoid-0-0-0)的博客，将其实现，主要原理请看[胡峻峥](https://www.cnblogs.com/hujunzheng/p/10353390.html#autoid-0-0-0)博客，下面将具体做法展示

# 开始

## DynamicTask动态任务

```java
import org.apache.commons.lang3.StringUtils;
import org.assertj.core.util.Lists;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.TaskScheduler;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.config.CronTask;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import org.springframework.scheduling.config.TriggerTask;
import org.springframework.scheduling.support.CronSequenceGenerator;
import org.springframework.scheduling.support.PeriodicTrigger;
import org.springframework.util.CollectionUtils;

import javax.annotation.PreDestroy;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;

/**
 * <p>
 * 动态定时任务
 * </P>
 *
 * @author bing_huang
 * @since V1.0
 */
@Configuration
public class DynamicTask implements SchedulingConfigurer {
    private static Logger LOGGER = LoggerFactory.getLogger(DynamicTask.class);
    private volatile ScheduledTaskRegistrar registrar;
    private ConcurrentHashMap<String, ScheduledFuture<?>> scheduledFutures = new ConcurrentHashMap<>();
    private ConcurrentHashMap<String, CronTask> cronTasks = new ConcurrentHashMap<>();

    private volatile List<TaskConstant> taskConstants = Lists.newArrayList();

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        this.registrar = registrar;
        this.registrar.addTriggerTask(() -> {
            if (!CollectionUtils.isEmpty(taskConstants)) {
                LOGGER.info("检测动态定时任务列表...");
                List<SchedulingRunnable> runnables = new ArrayList<>();
                taskConstants.forEach(taskConstant -> {
                    SchedulingRunnable runnable = new SchedulingRunnable(taskConstant.getTaskId(), taskConstant.getCron());
                    runnables.add(runnable);
                });
            }
        }, triggerContext -> new PeriodicTrigger(5L, TimeUnit.SECONDS).nextExecutionTime(triggerContext));
    }

    /**
     * <p>
     * 刷新
     * </p>
     */
    public void refreshTasks(List<SchedulingRunnable> runnables) {
        Set<String> taskIds = scheduledFutures.keySet();
        //取消已经删除的策略任务
        for (String taskId : taskIds) {
            if (!exists(runnables, taskId)) {
                scheduledFutures.remove(taskId).cancel(false);
                cronTasks.remove(taskId);
            }
        }
        for (SchedulingRunnable runnable : runnables) {
            String expression = runnable.getExpression();
            if (StringUtils.isBlank(expression) || !CronSequenceGenerator.isValidExpression(expression)) {
                LOGGER.error("定时任务DynamicTask cron表达式不合法: " + expression);
                continue;
            }
            //配置相同,则不需要重新创建定时任务
            if (scheduledFutures.containsKey(runnable.getTaskId())
                    && cronTasks.get(runnable.getTaskId()).getExpression().equalsIgnoreCase(expression)) {
                continue;
            }
            // 如果策略执行时间发生了变化，则取消当前策略的任务
            if (scheduledFutures.containsKey(runnable.getTaskId())) {
                scheduledFutures.remove(runnable.getTaskId()).cancel(false);
                cronTasks.remove(runnable.getTaskId());
            }
            CronTask task = new CronTask(runnable, expression);
            ScheduledFuture<?> future = registrar.getScheduler().schedule(task.getRunnable(), task.getTrigger());
            cronTasks.put(runnable.getTaskId(), task);
            scheduledFutures.put(runnable.getTaskId(), future);
        }
    }

    /**
     * <p>
     * 是否存在相同任务
     * </p>
     *
     * @param tasks  任务运行
     * @param taskId 任务id
     * @return 是否存在
     */
    private boolean exists(List<SchedulingRunnable> tasks, String taskId) {
        for (SchedulingRunnable task : tasks) {
            if (task.getTaskId().equals(taskId)) {
                return true;
            }
        }
        return false;
    }

    /**
     * <p>
     * 获取所有任务
     * </p>
     *
     * @return 任务集
     */
    public List<TaskConstant> getTaskConstants() {
        return taskConstants;
    }

    /**
     * <p>
     * 新增任务
     *
     * </p>
     *
     * @param taskId     任务id
     * @param expression 表达式
     */
    public void addTriggerTask(String taskId, String expression) {
        if (StringUtils.isBlank(expression) || !CronSequenceGenerator.isValidExpression(expression)) {
            LOGGER.error("定时任务DynamicTask cron表达式不合法: " + expression);
            return;
        }
        //配置相同,则不需要重新创建定时任务
        if (scheduledFutures.containsKey(taskId)
                && cronTasks.get(taskId).getExpression().equalsIgnoreCase(expression)) {
            return;
        }
        // 如果策略执行时间发生了变化，则取消当前策略的任务
        if (scheduledFutures.containsKey(taskId)) {
            scheduledFutures.remove(taskId).cancel(false);
            cronTasks.remove(taskId);
        }
        SchedulingRunnable runnable = new SchedulingRunnable(taskId, expression);
        CronTask task = new CronTask(runnable, expression);
        addTriggerTask(taskId, task);
    }

    /**
     * 新增任务
     *
     * @param taskId      任务id
     * @param triggerTask 触发器
     */
    public void addTriggerTask(String taskId, TriggerTask triggerTask) {
        TaskScheduler scheduler = registrar.getScheduler();
        assert scheduler != null;
        ScheduledFuture<?> schedule = scheduler.schedule(triggerTask.getRunnable(), triggerTask.getTrigger());
        assert schedule != null;
        cronTasks.put(taskId, (CronTask) triggerTask);
        scheduledFutures.put(taskId, schedule);
    }

    /**
     * <p>
     * 取消计划
     * </p>
     *
     * @param taskId 任务id
     */
    public void cancelTriggerTask(String taskId) {
        ScheduledFuture<?> future = scheduledFutures.get(taskId);
        if (future != null) {
            future.cancel(false);
        }
        scheduledFutures.remove(taskId).cancel(false);
        cronTasks.remove(taskId);
        List<TaskConstant> list = new ArrayList<>();
        taskConstants.forEach(taskConstant -> {
            if (taskConstant.getTaskId().equals(taskId)) {
                list.add(taskConstant);
            }
        });
        taskConstants.removeAll(list);
    }

    /**
     * <p>
     * 全部取消
     * </p>
     */
    public void cancelTriggerTask() {
        ConcurrentHashMap.KeySetView<String, ScheduledFuture<?>> keys = scheduledFutures.keySet();
        for (String key : keys) {
            cancelTriggerTask(key);
        }
    }

    /**
     * <p>
     * 销毁
     * </p>
     */
    @PreDestroy
    public void destroy() {
        this.registrar.destroy();
    }

    /**
     * <p>
     * 任务编号
     * </p>
     *
     * @return 任务id集
     */
    public Set<String> taskIds() {
        return scheduledFutures.keySet();
    }

    /**
     * <p>
     * 是否存在任务
     * </p>
     *
     * @param taskId 任务id
     * @return 是否存在
     */
    public boolean hasTask(String taskId) {
        return scheduledFutures.containsKey(taskId);
    }

}
```

## SchedulingRunnable

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Objects;

/**
 * <p>
 *     定时任务运行类
 * </P>
 *
 * @author bing_huang
 * @since V1.0
 */
public class SchedulingRunnable implements Runnable {
    private static final Logger logger = LoggerFactory.getLogger(SchedulingRunnable.class);
    private String expression;

    private String taskId;

    public SchedulingRunnable() {
    }

    public SchedulingRunnable(String expression, String taskId) {
        this.expression = expression;
        this.taskId = taskId;
    }

    public String getExpression() {
        return expression;
    }

    public void setExpression(String expression) {
        this.expression = expression;
    }

    public String getTaskId() {
        return taskId;
    }

    public void setTaskId(String taskId) {
        this.taskId = taskId;
    }

    @Override
    public void run() {
        logger.info("定时任务开始执行");
        long startTime = System.currentTimeMillis();
        long times = System.currentTimeMillis() - startTime;
        logger.info("定时任务执行结束 {} 毫秒", times);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        SchedulingRunnable that = (SchedulingRunnable) o;
        return Objects.equals(expression, that.expression) &&
                Objects.equals(taskId, that.taskId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(expression, taskId);
    }
}
```

## 动态任务TaskConstant

```java
package spring.scheduling.test.config;

import java.util.Objects;

/**
 * <p>
 * </P>
 *
 * @author bing_huang
 * @since V1.0
 */
public class TaskConstant {
    /**
     * <p>
     * 表达式
     * </P>
     */
    private String cron;
    /**
     * <p>
     * 任务id
     * </p>
     */
    private String taskId;

    public String getCron() {
        return cron;
    }

    public void setCron(String cron) {
        this.cron = cron;
    }

    public String getTaskId() {
        return taskId;
    }

    public void setTaskId(String taskId) {
        this.taskId = taskId;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        TaskConstant that = (TaskConstant) o;
        return Objects.equals(cron, that.cron) &&
                Objects.equals(taskId, that.taskId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(cron, taskId);
    }
}
```

主要是通过spring提供的一个任务调度`org.springframework.scheduling.annotation.SchedulingConfigurer`来实现动态的任务调度。

+ taskConstants 动态任务列表
+ ScheduledTaskRegistrar#addTriggerTask 添加动态周期定时任务，检测动态任务列表的变化
+ 动态创建cron定时任务，拿到ScheduledFuture实例并缓存起来
+ 在刷新任务列表时，通过缓存的ScheduledFuture实例和CronTask实例，来决定是否取消、移除失效的动态定时任务。

# thanks

[胡峻峥](https://www.cnblogs.com/hujunzheng/p/10353390.html#autoid-0-0-0)

# github

<https://github.com/hb0730/spring-boot-samples/tree/master/spring-boot-dynamictask>
