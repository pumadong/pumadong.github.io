---

layout: single
title: Java学习（定时任务）
permalink: /java/java-timer.html

classes: wide

author: Bob Dong

---

# 前言

在Java中，实现定时任务有多种方式，本文介绍4种，Timer和TimerTask、Spring、QuartZ、Linux Cron。

以上4种实现定时任务的方式，Timer是最简单的，不需要任何框架，仅仅JDK就可以，缺点是仅仅是个时间间隔的定时器，调度简单；Spring和QuartZ都支持cron，功能都很强大，Spring的优点是稍微简单一点，QuartZ的优点是没有Spring也可使用；Linux Cron是个操作系统级别的定时任务，适用于所有操作系统支持的语言，缺点是精度只能到达分钟级别。



Timer和TimerTask


关于Timer定时器的实现原理，如果我们看过JDK源码，就会发现，是使用的Object.wait(timeout)，来进行的线程阻塞，timeout是根据下次执行实际和当前实际之差来计算。实际上，这可以归结为一个多线程协作（协作都是在互斥下的协作）问题。

在java.util.concurrent中，有个ScheduledThreadPoolExecutor，也可以完全实现定时任务的功能。

而其他的框架，无非是功能的增强，特性更多，更好用，都是在基础的java之上的包装。

代码示例如下：

```
import java.util.Date;  
import java.util.Timer;  
import java.util.TimerTask;   
public class TimerTest extends TimerTask  
{  
    private Timer timer;       
    public static void main(String[] args)  
    {  
        TimerTest timerTest= new TimerTest();  
        timerTest.timer = new Timer();            
        //立刻开始执行timerTest任务，只执行一次  
        timerTest.timer.schedule(timerTest,new Date());            
        //立刻开始执行timerTest任务，执行完本次任务后，隔2秒再执行一次  
        //timerTest.timer.schedule(timerTest,new Date(),2000);            
        //一秒钟后开始执行timerTest任务，只执行一次  
        //timerTest.timer.schedule(timerTest,1000);            
        //一秒钟后开始执行timerTest任务，执行完本次任务后，隔2秒再执行一次  
        //timerTest.timer.schedule(timerTest,1000,2000);            
        //立刻开始执行timerTest任务，每隔2秒执行一次  
        //timerTest.timer.scheduleAtFixedRate(timerTest,new Date(),2000);           
        //一秒钟后开始执行timerTest任务，每隔2秒执行一次  
        //timerTest.timer.scheduleAtFixedRate(timerTest,1000,2000);  
 
        try  
        {  
            Thread.sleep(10000);  
        } catch (InterruptedException e)  
        {  
            e.printStackTrace();  
        }
        //结束任务执行，程序终止  
        timerTest.timer.cancel();  
        //结束任务执行，程序并不终止,因为线程是JVM级别的  
        //timerTest.cancel();  
    }    
    @Override  
    public void run()  
    {  
        System.out.println("Task is running!");  
    }  
}
```

使用spring @Scheduled注解执行定时任务
这种方式非常简单，却能使用cron完成和QuartZ一样的功能，值得推荐一下。



ApplicationContext.xml：



beans根节点增加内容：xmlns:task="http://www.springframework.org/schema/task"

beans根节点中，xsi:schemaLocation属性下增加:

http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task-3.1.xsd

<!-- 开启注解任务 -->

<task:annotation-driven/>

实现类：

```

@Component  //import org.springframework.stereotype.Component;  
public class MyTestServiceImpl  implements IMyTestService {  
      @Scheduled(cron="0/5 * *  * * ? ")   //每5秒执行一次  
      @Override  
      public void myTest(){  
            System.out.println("进入测试");  
      }  
}
```

注意几点：


spring的@Scheduled注解  需要写在实现上；

定时器的任务方法不能有返回值；

实现类上要有组件的注解@Component，@Service，@Repository



QuartZ


QuartZ With Spring

applicationContext-schedule.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xmlns:context="http://www.springframework.org/schema/context"
 xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd">
 <bean id="quartzScheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
  <property name="triggers">
   <list>
    <!-- 启动的Trigger列表 -->
    <ref local="startThriftTrigger" />
   </list>
  </property>
  <property name="quartzProperties">
   <props>
    <prop key="org.quartz.threadPool.threadCount">5</prop>
   </props>
  </property>
  <!-- 启动时延期3秒开始任务 -->
  <property name="startupDelay" value="3" />
  <property name="autoStartup" value="${scheduler.autoStartup}" />
 </bean>
 
 
 <!-- 启动Thrift，立即启动 -->
 <bean id="startThriftTrigger" class="org.springframework.scheduling.quartz.SimpleTriggerBean">  
  <property name="startDelay" value="0" />  
        <property name="repeatInterval" value="1000" />
        <property name="repeatCount" value="0" />
        <property name="jobDetail" ref="startThriftTask" /> 
 </bean>
 <bean id="startThriftTask" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
  <property name="targetObject" ref="startThrift" />
  <property name="targetMethod" value="execute" />
  <!-- 同一任务在前一次执行未完成而Trigger时间又到时是否并发开始新的执行, 默认为true. -->
  <property name="concurrent" value="true" />
 </bean>
</beans>
```

#### 实现类

```

package xx.schedule;
 
@Component
public class StartThrift {
 /**
  * 调度入口
  */
 public void execute() {
  // to do something
 } 
}
```

### QuartZ No Spring



**使用的QuartZ jar是1.8.5，如下：**

```
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>1.8.5</version>
</dependency>
```

**调用类实现代码如下：**

```
import org.quartz.CronExpression;
import org.quartz.CronTrigger;
import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.SchedulerFactory;
import org.quartz.impl.StdSchedulerFactory;
 
public class InvokeStatSchedule {   
    public void start() throws SchedulerException
    {
        SchedulerFactory schedulerFactory = new StdSchedulerFactory();
        Scheduler scheduler = schedulerFactory.getScheduler();
 
        //InvokeStatJob是实现了org.quartz.Job的类
        JobDetail jobDetail = new JobDetail("jobDetail", "jobDetailGroup", InvokeStatJob.class);
        CronTrigger cronTrigger = new CronTrigger("cronTrigger", "triggerGroup");
        try {
            CronExpression cexp = new CronExpression("0 0 * * * ?");
            cronTrigger.setCronExpression(cexp);
        } catch (Exception e) {
            e.printStackTrace();
        }        
        scheduler.scheduleJob(jobDetail, cronTrigger);
        scheduler.start();
    }   
}
```

**定时任务类代码如下：**

import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

public class InvokeStatJob implements Job {

    @Override
    public void execute(JobExecutionContext arg0) throws JobExecutionException {
        //...要定时操作的内容   
    }
}

## Linux Cron

这其实也是一种非常普遍的实现定时任务的方式，实际是操作系统的定时任务。Linux Cron只能到达分钟级，到不了秒级别。

一般我们是设置定时执行一个sh脚本，在脚本里面写一些控制代码，例如，有如下的脚本：

3,33 * * * * /usr/local/log_parser/run_log_parser.sh &



run_log_parser.sh的内容大致如下：

```
#!/bin/sh

log_parser_dir=/usr/local/log_parser
tmp_file=/usr/local/run_parser_tmp.txt
parser_log=/usr/local/access_parser.log
tmpDir=/data/applogs/access_logs_bp

date >> "$parser_log"


if [! -f "$tmp_file"]; then
        echo '访问日志解析正在进行,尚未完成' >> "$parser_log"
        echo '' >> "$parser_log"
else
        echo '开始解析访问日志' >> "$parser_log"
        touch "$tmp_file"
        cd "$log_parser_dir"
        python access_log_parser.py >> "$parser_log"
        rm "$tmp_file"
        echo '解析访问日志完成' >> "$parser_log"
        echo '' >> "$parser_log"
        cd "$tmpDir"
        gzip gzip WEB0*
        mv *.gz gz/
        echo '压缩备份日志及移动到压缩目录成功' >> "$parser_log"
fi
```



# 后记

