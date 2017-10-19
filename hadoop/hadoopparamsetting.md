# 参数设置

## yarn参数设置

| 参数名称 | 参数默认值 | 参数说明 |
| :--- | :--- | :--- |
| yarn.nodemanager.aux-services |   | 附属服务，需配置成mapreduce\_shuffle才可运行MapReduce程序 |
| yarn.resourcemanager.address | ${yarn.resourcemanager.hostname}:8032 | ResourceManager 对客户端暴露的地址。客户端通过该地址向RM提交应用程序，杀死应用程序等。 |
|yarn.resourcemanager.scheduler.address  | ${yarn.resourcemanager.hostname}:8030 | ResourceManager 对ApplicationMaster暴露的访问地址。ApplicationMaster通过该地址向RM申请资源、释放资源等
| yarn.resourcemanager.resource-tracker.address| ${yarn.resourcemanager.hostname}:8031|ResourceManager 对NodeManager暴露的地址.。NodeManager通过该地址向RM汇报心跳，领取任务等|
|yarn.resourcemanager.admin.address|${yarn.resourcemanager.hostname}:8033|ResourceManager 对管理员暴露的访问地址。管理员通过该地址向RM发送管理命令等。|
|yarn.resourcemanager.webapp.address|${yarn.resourcemanager.hostname}:8088|ResourceManager对外web ui地址。用户可通过该地址在浏览器中查看集群各类信息。|
|yarn.resourcemanager.scheduler.class|org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler|启用的资源调度器主类。目前可用的有FIFO、Capacity Scheduler和Fair Scheduler。|





