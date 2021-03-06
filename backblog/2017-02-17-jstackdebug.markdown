---
layout: post
title:  Jstack-DEBUG
date:   2017-01-17 21：34：23 +0800
categories: blog

---


背景介绍

1. 积分交易上线后，积分账户出现死锁（偶现）。死锁后，对应账户后续积分使用，都会锁等待超时。
2. 线上相关业务系统，都无错误日志。
3. 由于开发、QA环境过通、并发测试通过，一开始怀疑是由于并发引起的BUG死锁。
4. 大家认为是新业务BUG，相关人员一起review新业务代码，然而并没有找到问题。

随后我在服务上看，CPU资源消耗很高。平常20%以内，一但锁死出现后,CPU会逐渐增长。 70%, 120%, 160%。
又因为，一但死锁后，对于该账户续的锁都会等待超时,所以初步怀疑进入了死循环或者死锁。

后面通过jstack查看，有处执行栈代码很可疑，是生成业务ID的部分代码。因为生成ID很快。然而每次jstack都能捕获到该执行栈。
[贴图]
img/jstack/jstack-stack.png

我反馈给相应同事，他们认为不可能，理由也很简单。
1. 业务ID生成，使用了很久了，半年以上。
2. 其他业务也在使用这个基础组件。
3. 其他业务目前是正常工作的。
4. 之前QA上测试是正确的。

直觉告诉我80%都可能是那出现的错误，再拿到基础库地址后，查看对应代码逻辑，确认是这个问题导致的。在构建几个字符串发确认是必现。

{% highlight java lineanchors %}
public String randomizeNumericString(String input) {
    try{
        Long.parseLong(input);
    }catch (RuntimeException ex){
        throw ex;
    }
    char[] result = new char[input.length()];
    char[] chars = input.toCharArray();
    int count = 0;
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


[贴代码]
[贴构建代码]
原因是因为生成ID， 使用了时间,精确到秒。然后随机打乱数字，并且需要保证第前位数字必须大于0 和 小于等于4，否则continue;（历史原因规则)

2017-02-17T02:54:00.0Z 时间点时，(System.currentTimeMillis()/1000/30)正好全部数字为 49576668。很大机率都是这样的数字。
修复后总结了这次BUG原因。

为什么使用了这么严重问题，使用半年却看似正常？
1. 业务ID生成，使用了很久了，半年以上。
2. 其他业务也在使用这个基础组件。
3. 其他业务目前是正常工作的。
4. QA上是正确的。

原因: 
1.BUG为偶现，机率较小，其他业务不会使用到锁，出现该BUG时，该线程会“假死”掉。超时后再尝试会成功。
2.迭代速度较快，发布很频繁，QA每天build很多次， 线上2-3天发布一次。让所有线程因 偶现BUG“假死”掉之前，重启后又恢复正常。


1. 使用当前时间算法，可能随时间产生。
2. 使用jstack定位 CPU飙高问题，很有卓效。
3. 以实事为依据，不要完全由历史经验判断（这个东西用很久了）。