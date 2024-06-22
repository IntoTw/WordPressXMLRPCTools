---
title: 'Spring Boot Scheduled定时任务特性'
date: 2020-10-14
lastmod: 2020-10-14
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["spring","scheduled"]
categories: ["技术"]
lightgallery: true
---

SpringBoot中的Scheduled定时任务是Spring Boot中非常常用的特性，用来执行一些比如日切或者日终对账这种定时任务


下面说说使用时要注意的Scheduled的几个特性


## Scheduled的执行方式 
Scheduled按照顺序执行，对于某个task未做配置的话只会起一个线程去执行，也就是说当你某个任务在处理中阻塞了，哪怕轮询时间再次到达，Spring也不会再起线程执行该任务，而是会等待上次任务执行完毕，所以请不要在Scheduled的task中做一些比较需要频繁触发的易失败，易阻塞，易超时操作，避免任务无法正常轮询执行


## Scheduled中的线程池 
Scheduled执行可以通过Spring Boot提供的配置来配置定时任务执行的线程池等信息，如果未做配置的话，根据我测试，所有定时任务仅有一个线程去执行，也就是说如果某个task阻塞，其他task都将得不到执行。具体配置方法如下

	@Configuration
	@EnableScheduling
	public class SchedulerTaskConfiguration implements SchedulingConfigurer {
	    @Override
	    public void configureTasks(ScheduledTaskRegistrar scheduledTaskRegistrar) {
	        scheduledTaskRegistrar.setScheduler(taskExecutor());
	    }
	
	    @Bean
	    public Executor taskExecutor() {
	        //线程池大小
	        return Executors.newScheduledThreadPool(10);
	    }
	}

但是哪怕是配置了线程池，也只是降低了多个task之间执行的影响，对于单个task来说，哪怕配置了线程池，依旧会因为上次执行的阻塞影响到下一次触发

## Scheduled的执行频率

Scheduled的执行频率可以由2种方式控制，一种为在@Scheduled中添加fixedRate属性，即@Scheduled(fixedRate = 10)，数字为执行的间隔毫秒，也就是多少毫秒执行一次

另一种为添加cron属性，属性值为cron表达式，可以通过cron表达式指定为具体某年某月某分某秒，也可以通过cron表达式指定为间隔几小时或几分钟执行一次,如**@Scheduled(cron = "0 0 1 * * *")**
