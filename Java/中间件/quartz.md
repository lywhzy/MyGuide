# 定时器 Quartz

    定时器主要应用与定时处理任务，类似于Java中的定时任务线程池，只不过Quartz的功能更为强大，且在各个方面封装的十分优秀。
    通过专门的cron表达式来描述任务的执行周期，cron表达式规则如下图所示

![binaryTree](../image/微信图片_20201009164517.png) 

## 三大概念

- Job
- Trigger(触发器)
- Scheduler

## cron表达式示例

![binaryTree](../image/微信图片_20201009164916.png)

## SpringBoot集成Quartz

- 导入spring-boot-starter-quartz依赖
- 注入ScheduleFactoryBean 
- 通过ScheduleFactoryBean操作定时任务

### Scheduler

- pauseJob 暂停任务
````
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
        scheduler.getTriggerKeys(GroupMatcher.anyGroup()).forEach(triggerKey -> {
            try {
                Trigger trigger = scheduler.getTrigger(triggerKey);
                if(trigger.getKey().getName().equals(name)){
                    System.out.println(name + ": 已暂停");
                    scheduler.pauseJob(trigger.getJobKey());
                }
            } catch (SchedulerException e) {
                e.printStackTrace();
            }
        });
````
- resumeJob 恢复任务
````
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
        scheduler.getTriggerKeys(GroupMatcher.anyGroup()).forEach(triggerKey -> {
            try {
                Trigger trigger = scheduler.getTrigger(triggerKey);
                if(trigger.getKey().getName().equals(name)){
                    System.out.println(name + ": 已启用");
                    scheduler.resumeJob(trigger.getJobKey());
                }
            } catch (SchedulerException e) {
                e.printStackTrace();
            }
        });
````
- triggerJob 执行一次任务


### @DisallowConcurrentExecution

    @DisallowConcurrentExecution : 此标记用在实现Job的类上面,意思是不允许并发执行.
    注org.quartz.threadPool.threadCount的数量有多个的情况,@DisallowConcurrentExecution才生效
