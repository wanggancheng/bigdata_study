
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
|yarn.nodemanager.local-dirs|${hadoop.tmp.dir}/nm-local-dir|中间结果存放位置，类似于1.0中的mapred.local.dir。注意，这个参数通常会配置多个目录，已分摊磁盘IO负载。存放application执行本地文件的根目录，执行完毕后删除，按用户名存储|
|yarn.nodemanager.log-dirs|${yarn.log.dir}/userlogs|日志存放地址（可配置多个目录）存放application本地执行日志的根目录，执行完毕后删除，按用户名存储|
|yarn.nodemanager.log.retain-seconds|10800（3小时）|NodeManager上日志最多存放时间（不启用日志聚集功能时有效）|
|yarn.log-aggregation-enable|false|是否启用日志聚合功能，日志聚合开启后保存到HDFS上|
|yarn.nodemanager.remote-app-log-dir|/tmp/logs|当应用程序运行结束后，日志被转移到的HDFS目录（启用日志聚集功能时有效），修改为保存的日志文件夹|
|yarn.nodemanager.remote-app-log-dir-suffix|logs 日志将被转移到目录${yarn.nodemanager.remote-app-log-dir}/${user}/${thisParam}下|远程日志目录子目录名称（启用日志聚集功能时有效）|
|yarn.log-aggregation.retain-seconds|-1|聚合后的日志在HDFS上保存多长时间，单位为s。|
|yarn.log-aggregation.retain-check-interval-seconds|-1|删除任务在HDFS上执行的间隔，执行时候将满足条件的日志删除（超过参数2设置的时间的日志），如果是0或者负数，则为参数2设置值的1/10，上例值在此处为8640s|
|yarn.nodemanager.log.retain-seconds|10800|当不启用日志聚合此参数生效，日志文件保存在本地的时间，单位为s|
|yarn.log.server.url||log server的地址|






