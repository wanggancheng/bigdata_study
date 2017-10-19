# 参数设置

## yarn参数设置

| 参数名称 | 参数默认值 | 参数说明 |
| :--- | :--- | :--- |
| yarn.nodemanager.aux-services |   | 附属服务，需配置成mapreduce\_shuffle才可运行MapReduce程序 |
| yarn.resourcemanager.address | ${yarn.resourcemanager.hostname}:8032 | ResourceManager 对客户端暴露的地址。客户端通过该地址向RM提交应用程序，杀死应用程序等。 |
|yarn.resourcemanager.scheduler.address  | ${yarn.resourcemanager.hostname}:8030 | ResourceManager 对ApplicationMaster暴露的访问地址。ApplicationMaster通过该地址向RM申请资源、释放资源等
| | |




