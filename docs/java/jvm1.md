背景：

ANS 偶发CPU利用率100%，组件调用频繁返空。

分析思路：

1、 优先考虑新上功能。TM升级4.0,点赞点踩上线，FastJson升级1.2.60。

2、优先定位产生的大体原因。

分析步骤：

1、根据预警找到机器Ip。登录ai-swat.jd.com,连接终端。

2、 top 命令查到占用CPU最高的进程。

![top](/docs/java/images/jvm1_1.png)

3、  top –Hp 391101 查到本进程下占用CPU最高的线程。

![topHp](/docs/java/images/jvm1_2.png)

4、printf "%x\n" 391474     获取线程 16进制值 5f932

5、jstack 139813 |grep 5f932 –C 20 可打印出现问题的代码位置。 （未保留现场，以上图和数据均为模拟）

![error1](/docs/java/images/jvm1_3.png)

![error1](/docs/java/images/jvm1_4.png)

![error1](/docs/java/images/jvm1_5.png)

打印出的信息指向了Jdk1.7 hashmap中的465行。 （上面那行命令打印出的信息）。至此可以定位hashMap扩容时发生了环。CPU会在这里一直进行死循环，从而导致CPU占用高。

6、初步定位到原因后，查找代码中，哪里会对requestContext中的context进行put操作。反复确定，没有对其进行操作。进行了一些优化操作，未果。

7、排查异常答案时，偶然发现，调用TM很小的概率会抛出异常。对异常日志进行分析。定位到是TM在调用JSF服务时，序列化异常。进一步分析，是ConcurrentModificationException异常！！！并发调用修改！！！

追求进TMSDK的代码，发现了对RequestContext的操作!!!

![error1](/docs/java/images/jvm1_6.png)

因为TM本地化后，对Rc的操作就变成了本地进行。 而ANS对TM的调用，本地是并发进行的。 任何一次对Rc的修改都可能会引发Rc中Context的扩容。当Rc中的Context达到阈值需要扩容时，会进行transfer操作，可能会形成环，导致CPU占用率100%。 当一条线程进行jsf序列化时，另一条线程对rc进行了修改，就会引发ConcurrentModificationException异常。

解决方案：

TM新增方法参数，本地不对RequestContext进行更新， RequestContext的操作放到远程去做，不影响上游应用的调用。

导出heap文件

jmap -dump:live,format=b,file=/export/Logs/heap.dump  pid