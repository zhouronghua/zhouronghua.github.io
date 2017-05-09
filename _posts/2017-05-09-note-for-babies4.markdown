---
layout: post
title:  "奶爸笔记之四"
date:   2017-03-18 22:32:23 +0800
categories: note for babies
---


你以为宝宝睡觉是个if else：
{% highlight c linenos %}
if(要睡觉){
sleep(3小时)
}
else{
陪我玩()
}

{% endhighlight %}
其实有是个while循环：
{% highlight c linenos %}
while(要睡觉&&有人抱&&抱的人认识){
sleep(30秒)
if(睡熟了){
sleep(3小时)
}
}
陪我玩()

{% endhighlight %}
更复杂的还要加个等待信号：
{% highlight c linenos %}
while(要睡觉&&有人抱&&抱的人认识&&没其他吵闹信号){
sleep(30秒)
if(睡熟了){
sleep_on(3小时,没其他吵闹信号)
}
}
if(!自然醒){
哭闹(半分钟)
}
陪我玩()
{% endhighlight %}
