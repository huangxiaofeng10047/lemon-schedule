# lemon-schedule

## 项目介绍
    
lemon-schedule是一个基于akka的轻量级分布式作业调度框架，其核心设计目标是高可用、分布式、轻量级、易扩展，核心功能是cron定时作业、作业实例间任务依赖的管理等。
该框架采用Scala语言开发，底层基于akka，目前属于实验性质的项目，欢迎大家试用。

## 软件架构

lemon-schedule有以下4中节点：

* Client：主要负责提交、取消作业，查询作业执行结果
* Manager：负责接收并分配作业，进行负载均衡
* Scheduler：负责作业调度，生成任务执行计划表、分发任务、聚合任务状态
* Worker：负责执行任务，失败重试，汇报任务执行结果

## 软件特点

1. cron作业定时调度。
2. 任务依赖。常见的调度系统都是指DAG图，即作业的依赖。用DAG图实现的依赖控制，其粒度是作业级别，即相互依赖的作业其执行频率是相同的。
    而lemon-schedule的“依赖”是任务的依赖，即作业实例间的依赖。如果B作业依赖于A，则B的某个实例执行时，会根据配置查找A作业某个实例的执行状态。
    例如A作业每小时执行一次，B作业每两个小时执行一次，如果B依赖于A作业前一个小时的任务实例，则当B作业12点执行时，会查找A作业11点任务
    的执行结果，若成功，则B运行，否则等待。
3. 无单点故障。所有类型的节点都是集群。
4. 指定节点运行。某些特殊情况下，例如作业执行的资源分布在特定机器，需要将作业限定在某些节点。
5. 失败重试。任务执行失败可重试指定次数。
6. 超时失败。任务可指定最长执行时间，超过该时间后被自动kill，进行重试。
7. 并发控制。作业可指定并行参数，即同时执行的任务个数。
8. 全异步。系统底层基于AKKA，数据库操作采用slick，实现全部异步化。

## TODO

1. 完善常见的任务。CommandTask、HiveTask等
2. 日志归集，便于排错
3. 完善未完成功能
4. 修改akka消息的序列化为kryo
5. 补充单元测试
6. 某个scheduler宕掉后，其他节点可以接管其工作。考虑使用集群分片实现，分片个数最好为节点个数的10倍
7. 使用ClusterRouterGroup（集群感知路由器）感知各个角色的节点，取代手工采集各个Actor信息
8. 减少消息体大小。避免actor之间传递过多冗余的数据

## 特殊说明

1. “作业”与“任务”，作业是调度的基本单位，是指静态信息；任务是指作业执行的具体实例。可参照“程序”、“进程”的关系进行理解。
2. lemon-schedule存在的目的有两个基本作用： 

    (1). 增加任务依赖的功能，弥补所有开源任务调度框架的功能缺失。
    
    (2). 用scala/akka实现一定的功能，给大家介绍一个先进的编程语言和并发框架。
    
3. 默认配置下lemon-schedule仅支持分钟级别的作业调度，如果希望支持秒级的作业调度，请修改相关参数配置。
   不过强烈建议不要支持秒级的作业。原因有两个：
   
   (1). 秒级的作业属于实时计算的范畴。不应该用作业调度实现。
   
   (2). 开启支持秒级的作业，调度器的频率也应该是秒级，此时对CPU会造成一定压力

## 安装教程

1. 执行packageAll.sh
2. 将distribute目录打包上传到服务器
3. 执行distribute/startManager.sh启动manager节点
4. 执行distribute/startScheduler.sh启动schedule节点
5. 执行distribute/startWorker.sh启动worker节点

## 使用说明

1. 参照lemon-schedule-examples开发Task
2. 将打包后jar包复制到distribute/lib按照“安装教程”进行部署
3. 通过Client客户端提交作业 

## 参与贡献

1. Fork 本项目
2. 新建 Feat_xxx 分支
3. 提交代码
4. 新建 Pull Request

## 遇到的问题和解决方案

1. 批量插入JOB
    
    批量处理JOB插入时，需要缓存未提交的JOB信息及其与原命令消息的对应关系，这样才能将插入成功的消息回复给原有的reply。
    但这比较占用内存，原有的消息体可能比较大；另外由于JOB的ID是数据库自增键，批量插入时是无法获取到该ID值的。
    所以JOB的ID值需要有程序自动生成，但这又涉及到全局唯一ID值生成规则。
    目前想到的全局唯一ID生成方案是IP+PORT+序列号，但节点重启后IP+PORT有肯能重复，所以进一步优化后的生成方案是：
    IP+PORT+STARTUP_TIMESTAMP+序列号。考虑到Actor模型的特殊性，其中的序列号生成不存在并发的问题，所以简单的计数器即可。
    但是IP和PORT的获取就有点麻烦了。
    所有的表增加UID字段，用来在业务上唯一标志数据，原有的自增ID不再使用，仅仅当做物理ID使用
    
## 联系作者

* QQ邮箱： echo "MzY1NzgxMDYyQHFxLmNvbQ==" | base64 -d
* 微信号： echo "SGVsbG9HcmFwZQo=" | base64 -d
* 手机号： echo "5L2g6KeJ5b6X5oiR5Lya57uZ5L2g5LmIXl9eCg==" | base64 -d