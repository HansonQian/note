# 什么是Quartz

​	`Quartz`是`OpenSymphony`开源组织在`Job Scheduling`领域的开源项目，它可以与`J2EE`、`J2SE`应用程序相结合，也可以单独使用。

​	`Quartz`是一个任务日程管理系统，一个在预先确定（被纳入日程）的时间到达时，负责执行（或者通知）其他软件组件的系统。

​	`Quartz`允许程序开发人员根据时间的间隔来调度作业。

​	`Quartz`实现了作业和触发器的多对多的关系，还能把多个作业与不同的触发器关联。

# Quartz下载与安装

## 下载

官方网站：<http://quartz-scheduler.org/>

## Maven依赖

一般使用`Quartz`是和`Spring`框架整合使用的，所以加入以下依赖

```xml
<dependency>
	<groupId>org.quartz-scheduler</groupId>
	<artifactId>quartz</artifactId>
	<version>2.2.2</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <version>4.1.3.RELEASE</version>
</dependency>
```

# Quartz架构

## Quartz框架的核心对象

- Scheduler： 代表一个调度容器，一个调度容器中可以注册多个 JobDetail 和 Trigger。当 Trigger 与 JobDetail 组合，就可以被 Scheduler 容器调度了

- Job：表示一个工作，要执行的具体内容。此接口中只有一个方法

  ```java
  void execute(JobExecutionContext context);
  ```

- JobDetail：表示一个具体的可执行的调度程序，Job 是这个可执行程调度程序所要执行的内容，另外 JobDetail 还包含了这个任务调度的方案和策略。

- Trigger：代表一个调度参数的配置，什么时候去调

## Quartz对象之间关系

![Scheduler](https://github.com/HansonQian/note/blob/master/imgs/Scheduler.png?raw=true)

## Quartz入门程序

### 案例

- 要去 ：每隔两秒打印一次Hello Quartz

- 步骤一引入Jar包

- 创建Job的实现类HelloJob，并实现execute方法

  ```java
  public class HelloJob implements Job {
      @Override
      public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
          DateTime dateTime = new DateTime();
          System.out.println("当前时间：" + dateTime.toString("yyyy-MM-dd HH:mm:ss"));
          System.out.println("Hello Quartz");
      }
  }
  ```

- 创建调度类,按照一定的策略执行上述的任务

  ```java
  public class QuartzExampleApp {
      public static void main(String[] args) throws SchedulerException {
          // 创建一个JobDetail的实例，将该实例与HelloJob绑定
          JobDetail jobDetail = JobBuilder.newJob(HelloJob.class).withIdentity("helloJob").build();
          // 创建一个Trigger触发器实例,定义改Job立即执行,并且没两秒执行一次持续执行
          SimpleTrigger simpleTrigger = TriggerBuilder.newTrigger()
                  .withIdentity("helloTrigger")
                  .startNow()// 立即执行
                  .withSchedule(SimpleScheduleBuilder
                          .simpleSchedule()// 简单的调度实现
                          .withIntervalInSeconds(2)// 定时器每隔2秒执行下去
                          .repeatForever()// 持续执行
                  )
                  .build();
          // 创建schedule
          SchedulerFactory schedulerFactory = new StdSchedulerFactory();
          Scheduler scheduler = schedulerFactory.getScheduler();
          scheduler.start();
          scheduler.scheduleJob(jobDetail, simpleTrigger);
      }
  }
  ```

- 运行结果

  ```powershell
  当前时间：2019-07-18 09:46:33
  Hello Quartz
  当前时间：2019-07-18 09:46:34
  Hello Quartz
  当前时间：2019-07-18 09:46:36
  Hello Quartz
  当前时间：2019-07-18 09:46:38
  Hello Quartz
  ```

- 总结

  - 创建调度工厂();  //工厂模式
  - 根据工厂取得调度器实例(); //工厂模式
  - Builder模式构建子组件<Job,Trigger>  //builder模式, 如JobBuilder、TriggerBuilder等
  - 通过调度器组装子组件   调度器.组装<子组件1,子组件2...>  //工厂模式
  - 调度器.start(); //工厂模式