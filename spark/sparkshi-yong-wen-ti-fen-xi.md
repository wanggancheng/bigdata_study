### spark on yarn提交任务时报ClosedChannelException解决方案
**问题描述**
spark2.1出来了，想玩玩就搭了个原生的apache集群，但在standalone模式下没有任何问题，基于apache hadoop 2.7.3使用spark on yarn一直报这个错。(Java 8)

报错日志如下：


```java
17/01/31 00:12:25 ERROR client.TransportClient: Failed to send RPC 8305478367380188725 to /192.168.56.101:43246: java.nio.channels.ClosedChannelException
java.nio.channels.ClosedChannelException
        at io.netty.channel.AbstractChannel$AbstractUnsafe.write(...)(Unknown Source)
17/01/31 00:12:25 INFO storage.BlockManagerMasterEndpoint: Trying to remove executor 1 from BlockManagerMaster.
17/01/31 00:12:25 INFO storage.BlockManagerMasterEndpoint: Removing block manager BlockManagerId(1, node01, 53420, None)
17/01/31 00:12:25 INFO storage.BlockManagerMaster: Removed 1 successfully in removeExecutor
17/01/31 00:12:25 INFO scheduler.DAGScheduler: Shuffle files lost for executor: 1 (epoch 0)
17/01/31 00:12:25 WARN cluster.YarnSchedulerBackend$YarnSchedulerEndpoint: Attempted to get executor loss reason for executor id 1 at RPC address 192.168.56.101:43252, but got no response. Marking as slave lost.
java.io.IOException: Failed to send RPC 8305478367380188725 to /192.168.56.101:43246: java.nio.channels.ClosedChannelException
        at org.apache.spark.network.client.TransportClient$3.operationComplete(TransportClient.java:249)
        at org.apache.spark.network.client.TransportClient$3.operationComplete(TransportClient.java:233)
        at io.netty.util.concurrent.DefaultPromise.notifyListener0(DefaultPromise.java:514)
        at io.netty.util.concurrent.DefaultPromise.notifyListenersNow(DefaultPromise.java:488)
        at io.netty.util.concurrent.DefaultPromise.access$000(DefaultPromise.java:34)
        at io.netty.util.concurrent.DefaultPromise$1.run(DefaultPromise.java:438)
        at io.netty.util.concurrent.SingleThreadEventExecutor.runAllTasks(SingleThreadEventExecutor.java:408)
        at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:455)
        at io.netty.util.concurrent.SingleThreadEventExecutor$2.run(SingleThreadEventExecutor.java:140)
        at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:144)
        at java.lang.Thread.run(Thread.java:745)
Caused by: java.nio.channels.ClosedChannelException
        at io.netty.channel.AbstractChannel$AbstractUnsafe.write(...)(Unknown Source)
17/01/31 00:12:25 ERROR cluster.YarnScheduler: Lost executor 1 on node01: Slave lost
17/01/31 00:12:26 INFO cluster.YarnSchedulerBackend$YarnSchedulerEndpoint: ApplicationMaster registered as NettyRpcEndpointRef(null)
17/01/31 00:12:26 INFO cluster.YarnClientSchedulerBackend: Add WebUI Filter. org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter, Map(PROXY_HOSTS -> node01, PROXY_URI_BASES -> http://node01:8088/proxy/application_1485792095366_0003), /proxy/application_1485792095366_0003
17/01/31 00:12:26 INFO ui.JettyUtils: Adding filter: org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter
17/01/31 00:12:28 ERROR cluster.YarnClientSchedulerBackend: Yarn application has already exited with state FINISHED!
```

