---
layout: post
title:  技术选型与落地方案 - 分布式锁
date:   2018-07-08 14：32：32 +0800
categories: blog
keywords: 分布式锁,Redis,Lock,Distributed Lock

---

如何做技术选型是比较大的一个话题，这篇Blog通过一个简单且实用的例子分享给大家，即分布式锁。
分布式锁在集群环境中，是逃不掉一环，并且比较全面而小巧，非常合适。

### 明确目标 - 避免解决不存在的问题
- 解决“内存锁”仅单节点有效，集群环境中无法协同生效的问题。
- 统一标准使用锁的方式，从设计上避免死锁。
- 统一命名规范，从设计上杜绝错误锁命名，以便管理(如:知道有多少处业务使用到了锁)。
- 简化配置，方便集成使用。
- 监控所有系统中使用锁的情况。

### 技术选型 - 有理有据

| 选型比较 | Redis | ZooKeeper | 备注 |
| ------- | ---- | ----------- | -- |
| 学习成本 | 低 | 相对高 | Redis 易调试、客户端UI友好、Telnet直接调试。ZooKeeper客户端简陋。|
| 运维成本 | 低 | 相对高 | 公司已经使用Redis已经很熟悉了，无需考虑 ZooKeeper 高可用方案 |
| 生态环境 | 一般 | 好 | Redis职责单一，但可定制化高也容易。ZooKeeper Java生态环境较好 Apache系很多项目集成ZooKeeper。 |
| 社区活跃 | 非常高 | 一般 | Redis： Watch: 2,404  Star 27,434  Fork 10,521 （统计截止2018-02-07，主要社区维护） <br /> ZooKeeper： Watch: 540  Star: 3,933  Fork: 2,769 (统计截止2018-02-07，主要有由Apache维护) |
| 个人倾向 | 倾向 | 不倾向 | 个人同类型中间件，选型标准：<br /> 1. 功能职责单一、易用、易调试 （Redis胜出）<br/> 2. 文档完善简洁易懂（Redis胜出） <br /> 3. 尽可能减少中间件依赖，Redis已投入使用 (Redis胜出) <br /> 4. 开发语言倾向: C 优于 Java (Redis胜出) <br /> 5. 自主官网维护优于Apache 系 (Redis胜出)|

### 从设计上防止死锁需满足条件 - 列出关键点

| 所需条件 | 解决方法 |
| ------- | ------- |
| 避免获取锁长时间等待 | 仅提供一种获取锁的方式tryLock，设置等待时间（小于等于5秒）。客户端需处理获取锁失败的情况 |
| 避免因Bug或Crash导致锁未释放 | 	获取锁【必须】指定解锁超时时间（最多30秒）|
| 获取锁必须是原子性操作(Atomic) | Redis [SETNX](https://redis.io/commands/setnx){:target="_blank"} 配合过期失效 [SET NX PX](https://redis.io/commands/set){:target="_blank"} |


### 画时序图 - 整理核心流程

![](/img/distributed-lock/dlock.svg)

### 代码示例 - 提供语法糖，简易使用。

{% highlight java lineanchors %}

@EnableDLock //启用分布式锁，自动配置
@SpringBootApplication
public class Application {
    
    //注入分布锁式组件
    @Autowired
    private LockManager lockManager;
	
    //使用demo，自动获取锁，自动释放，自动上报锁使用情况到监控
    @PostConstruct
    private void demo() {
        String category = "meeting-room";
        String business = "booking";
        String meetingRoomId = "1";
        int waitMillSeconds = 5000;
        String reuslt = lockManager.locker(category, business, meetingRoomId)
                // 获取锁成功后执行 - 业务代码
                .success(() ->  "预定会议成功。")
                //获取锁失败或等待超时后执行， 提示
                .failure(() ->  {throw new BusinessException("当前会议室正在预定中");})
                // 执行,愿意等待时长至多5秒（可选）
                .execute(waitMillSeconds);
    }	
	
	
    public static void main(String[] args) {
        SpringApplication.run(DlockDemoApplication.class, args);
    }
}

{% endhighlight %}

### 命名规范

- 分布式锁命名由：业务系统（spring.application.name）、子业务、子业务分类、业务ID组成
- 命名均为字符串并且小写以短杆(-)分隔，杜绝驼峰。例子：smart-office 反例：SmartOffice、smartOffice

### 使用规范
- 尽可能从设计上避免使用分布式锁，如乐观锁。
- 尽量减少锁的范围，尽快释放。
- 尝试获取锁超时，需要考虑正确反馈，如：“会议室预定繁忙，请稍后再试”。
- 禁止交叉锁，避免导致短暂的死锁，如：methodA: { a.tryLock(); b.tryLock(); } methodB:{ b.tryLock(); a.tryLock(); }
- 解锁超时时间至多30秒，超过考虑换设计方案。
- 等待获取锁时间至多5秒，超过则需要重试，或考虑其他设计方案。