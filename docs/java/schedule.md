[分布式定时任务调度系统技术选型](http://www.expectfly.com/2017/08/15/%E5%88%86%E5%B8%83%E5%BC%8F%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1%E6%96%B9%E6%A1%88%E6%8A%80%E6%9C%AF%E9%80%89%E5%9E%8B/)
[XXL-JOB原理--定时任务框架简介（一）](https://tianjunwei.blog.csdn.net/article/details/82595948?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-3.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-3.control)  

##java有哪些定时任务的框架

### 单机
1. timer
2. ScheduledExecutorService
3. spring定时框架

### 分布
1. Quartz
2. TBSchedule
3. elastic-job*
4. Saturn
5. xxl-job*

### quartz

### xxl-job
执行节点启动时注册到调度节点（数据库），
调度节点支持任务管理（前端），通过时间轮触发，调用执行节点执行任务。
执行节点通过反射执行指定方法。

### elastic-job

