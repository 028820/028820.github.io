---
layout: post
title:  Jstack-DEBUG
date:   2017-02-20 21：34：23 +0800
categories: blog
keywords: jstack,jcmd,debug

---


背景介绍

1. 积分交易上线后，积分账户偶尔会出现死锁，死锁后的账户再次使用积分消费都会锁等待超时。
2. 相关业务系统无错误日志。
3. 上线前开发、QA环境、并发性能测试通过。最初怀疑是由于并发引起的BUG死锁。
4. 大家认为是新业务BUG导致，于是相关人员一起Review新代码，然而并没有发现问题。

随后服务器监控上看见CPU负载异常，平常20%以内，锁死出现后CPU负载逐渐增长依次70%, 120%, 160%。
因为CPU负载增长后，对于部分用户积分消耗操作会超时，所以初步怀疑首次扣积分时锁未释放或代码BUG进入了无限重试。

随后通过jstack查看，发现有处执行栈很可疑。即生成业务ID的部分代码部分。<b>因为通常生成ID很快，jstack不太可能正好捕获到该执行栈</b>，然而每次jstack都能捕获到该执行栈。又因为CPU负载异常，所以判断是生成ID的模块有死循环BUG。

![](/img/jstack/jstack-stack.png)

我反馈给相应同事，他们认为不可能，理由也很简单。
1. 业务ID生成模块使用了很久了，半年以上。
2. 其他业务也在使用这个基础库。
3. 其他业务目前是正常工作的。
4. 之前QA上测试是正确的。

确实是他们说的这样，但是直觉告诉我是这里的问题，查看对应代码逻辑。最终确认是这个问题导致的，随后构建了几个字符串，确认是可复现BUG。
（历史遗留代码，不吐槽代码以及设计问题，聚焦分析问题过程)

{% highlight java lineanchors %}
public String randomizeNumericString(String input) {
    try{
        Long.parseLong(input);
    }catch (RuntimeException ex){
        throw ex;
    }
    char[] result = new char[input.length()];
    char[] chars = input.toCharArray();
    int count = 0;//下标
    int[] chosenIndexes = new int[input.length()];
    fillIntArray(chosenIndexes, -1);
    Random random = new Random(System.currentTimeMillis());
    while (count < result.length){
        int index = random.nextInt(chars.length);
        char c = chars[index];
        
        if((count == 0 && c == '0') || IntStream.of(chosenIndexes).anyMatch(x -> x == index)){
            continue;
        }

        if((count == 0 || count == 1) && Integer.parseInt(c+"") > 4){
            continue;
        }
        chosenIndexes[count] = index;
        result[count] = c;
        count ++;
    }
    return new String(result);
}
{% endhighlight %}


原因是生成ID时，使用了当前时间作为参数，精确到秒。然后随机打乱数字位置，前两位字符串必须于小等于4，并且第首位大于0，否则继续执行;


于是在 “2017-02-17T02:54:00.0Z”这个时间点左右，(System.currentTimeMillis()/1000/30)正好全部数字为49576668 （input参数）。很大机率都匹配“[1-9][5-9]+”，便一直进入循环状态。

修复后便总结这次BUG原因，为什么这么严重问题，使用半年却看似正常？

原因: 
1. BUG为偶现，机率较小，其他业务不会使用到锁，出现该BUG时，当前这个线程会“假死”掉，其他线程仍可以工作。超时后再尝试会成功。
2. 日常迭代速度较快，发布很频繁，QA每天build很多次， 线上2-3天发布一次。在所有线程因偶现BUG“假死”掉之前，发布后又恢复正常。

定位方法：
1. 使用jstack定位，CPU飙高问题，很有卓效。
2. 以实事为依据，不要完全由历史经验判断（如：这个东西已经使用很久了）。
