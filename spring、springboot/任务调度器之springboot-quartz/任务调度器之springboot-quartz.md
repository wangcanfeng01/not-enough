# 引入quartz
在springboot2.x以上版本中，都可以很方便的引入quartz，如下，在pom中增加一个dependency
``` xml
	 <dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-quartz</artifactId>
	</dependency>
```
# 配置Scheduler和SchedulerFactory
我在这里只是最简单的介绍一下springboot中quartz怎么使用，因此省略了很多复杂的步骤，如果想了解的更深入的话可以联系我活着翻阅源码
``` java
import org.quartz.Scheduler;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.scheduling.quartz.SchedulerFactoryBean;

import java.io.IOException;

/**
 * @author wangcanfeng
 * @description 配置scheduler工厂和scheduler，这里可以注入预先准备好的executor
 *或者使用系统默认的executor，都很方便，也可以把一些关键的参数放置在配置文件中，通过配置文件加载
 * @Date Created in 11:58-2019/3/15
 */
@Configuration
public class SchedulerAutoConfiguration {

    //这个地方如果需要使用自定义的executor，可以在别的地方配置好，然后这里注入
    //@Autowired
    //private ThreadPoolTaskExecutor taskExecutor;


    @Bean(name="SchedulerFactory")
    public SchedulerFactoryBean schedulerFactoryBean() throws IOException {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        factory.setAutoStartup(true);
       //这里如果不配置任务池，它就会默认加载SimpleThreadPool
       //factory.setTaskExecutor();
        return factory;
    }

    @Bean(name="funnyScheduler")
    public Scheduler scheduler() throws IOException {
        return schedulerFactoryBean().getScheduler();
    }
}
```
# 声明任务处理的curd接口
``` java
/**
 * @author wangcanfeng
 * @description
 * @Date Created in 13:53-2019/3/15
 */
public interface JobScheduleService {

    /**
     * 功能描述: 添加简单任务
     * @param  可以自定义一个任务信息对象，然后从信息对象中获取参数创建简单任务
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    void addSimpleJob();

    /**
     * 功能描述: 添加定时任务
     * @param 可以自定义一个任务信息对象，然后从信息对象中获取参数创建定时任务
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    void addCronJob();

    /**
     * 功能描述: 修改任务Trigger，即修改任务的定时机制
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    void modifyJob();

    /**
     * 功能描述: 暂停任务，只支持定时任务的暂停，不支持单次任务，单次任务需要interrupt
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    void pauseJob();

    /**
     * 功能描述: 从暂停状态中恢复定时任务运行
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    void resumeJob();

    /**
     * 功能描述: 删除任务
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    void deleteJob();

}
```
# 定义任务Class
``` java
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;


/**
 * @author wangcanfeng
 * @description
 * @Date Created in 13:49-2019/3/15
 */
public class SimpleJob implements Job {

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
            System.out.println("简单任务执行中");
    }
}
```
# 任务处理的接口实现
``` java
import org.quartz.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;
import org.springframework.util.ObjectUtils;

import java.util.Date;

/**
 * @author wangcanfeng
 * @description
 * @Date Created in 14:02-2019/3/15
 */
@Service
public class JobScheduleServiceImpl implements JobScheduleService {

    /**
     * 因为在配置中设定了这个bean的名称，这里就需要指定bean的名称，不然启动就会报错
     */
    @Autowired
    @Qualifier("funnyScheduler")
    private Scheduler scheduler;

    /**
     * 功能描述: 添加简单任务
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    @Override
    public void addSimpleJob(){
        JobDetail jobDetail = JobBuilder.newJob(SimpleJob.class)
                .withIdentity("testJob01", "testJob")
                .build();
        Date date=new Date();
        date.setTime(1552636529000l);
        SimpleTrigger simpleTrigger = (SimpleTrigger) TriggerBuilder.newTrigger()
                .withIdentity("testTrigger01", "testTrigger")
                .startAt(date)
                .build();
        try {
            scheduler.scheduleJob(jobDetail,simpleTrigger);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    /**
     * 功能描述: 添加定时任务
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    @Override
    public void addCronJob() {
        JobDetail jobDetail = JobBuilder.newJob(SimpleJob.class)
                .withIdentity("testJob01", "testJob")
                .build();
        //表达式调度构建器(即任务执行的时间)
        CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule("* * * * * ? *");
        //按新的cronExpression表达式构建一个新的trigger
        CronTrigger trigger = TriggerBuilder.newTrigger().
                withIdentity("cronTrigger01","cronTrigger")
                .withSchedule(scheduleBuilder)
                .build();
        try {
            scheduler.scheduleJob(jobDetail,trigger);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    /**
     * 功能描述: 修改任务Trigger，即修改任务的定时机制
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    @Override
    public void modifyJob() {
        TriggerKey oldKey=new TriggerKey("testTrigger01","testTrigger");
        //表达式调度构建器(即任务执行的时间)
        CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule("* * * * * ? *");
        //按新的cronExpression表达式构建一个新的trigger
        CronTrigger trigger = TriggerBuilder.newTrigger().
                withIdentity("cronTrigger01","cronTrigger")
                .withSchedule(scheduleBuilder)
                .build();
        try {
            scheduler.rescheduleJob(oldKey,trigger);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    /**
     * 功能描述: 暂停任务，只支持定时任务的暂停，不支持单次任务，单次任务需要interrupt
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    @Override
    public void pauseJob() {
        JobKey jobKey=new JobKey("testJob01","testJob");
        try {
            JobDetail jobDetail=  scheduler.getJobDetail(jobKey);
            if(ObjectUtils.isEmpty(jobDetail)){
                System.out.println("没有这个job");
            }
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
        try {
            scheduler.pauseJob(jobKey);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    /**
     * 功能描述: 从暂停状态中恢复定时任务运行
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    @Override
    public void resumeJob() {
        JobKey jobKey=new JobKey("testJob01","testJob");
        try {
            scheduler.resumeJob(jobKey);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    /**
     * 功能描述: 删除任务
     * @param
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/3/15 17:00
     */
    @Override
    public void deleteJob() {
        JobKey jobKey=new JobKey("testJob01","testJob");
        try {
            scheduler.deleteJob(jobKey);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }
}
```
然后就可以在controller中注入service，然后调用service中的方法，对任务进行curd操作了，是不是很简单呢，喜欢的话伸出你的点赞手，点个赞吧，如果有时间的话能去我的个人网站晃一圈就更好了
http://www.canfeng.xyz