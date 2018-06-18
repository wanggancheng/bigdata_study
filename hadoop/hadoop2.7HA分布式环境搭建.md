# Hadoop2.7.X HA分布式集群环境搭建

## 环境介绍

Linux系统：Centos 7.3 (64位)

内存：5GB

Hadoop版本：2.7.4

JDK版本：Oracle jdk1.8.0_121

ZooKepper:3.4.9



## Hadoop集群结构说明

## 集群结构

|       IP       |  主机名  |               Hadoop jps结果               |
| :------------: | :---: | :--------------------------------------: |
| 192.168.56.101 | node1 | NameNode/ResourceManager/ZKFC/DataNode/NodeManager/JournalNode/<br/>QuorumPeerMain |
| 192.168.56.102 | node2 | NameNode/ResourceManager/ZKFC/DataNode/NodeManager/JournalNode/<br/>QuorumPeerMain |
| 192.168.56.103 | node3 | DataNode/NodeManager/JournalNode/<br/>QuorumPeerMain |

## 环境准备(所有机器准备)

### /etc/hosts（需使用管理员权限）

```tex
192.168.56.101 node1
192.168.56.102 node2
192.168.56.103 node3
```

### 系统用户（需使用管理员权限）

```shell
sudo groupadd -g 10004  data
sudo useradd -u 10004 -g data -m data
echo "data"|sudo passwd --stdin data
```

### 用户data的ssh免密码登录

```shell
ssh-keygen  -t rsa 
#连敲三次回车
#把公钥传输到每个集群
ssh-copy-id node1
ssh-copy-id node2
ssh-copy-id node3
#验证能否保证用户ssh免密登录到其它机器
ssh node1
ssh node2
ssh node3
```



### 配置用户的java环境变量(~/.bashrc)

```shell
export JAVA_HOME="/opt/java/java"
export PATH=${JAVA_HOME}/bin:$PATH
```

### 创建一些目录

```shell
 mkdir -p ~/soft
 mkdir -p ~/app
 mkdir -p ~/data/dfs/namenode
 mkdir -p ~/data/dfs/datanode
 mkdir -p ~/data/zk/data
 mkdir -p ~/data/zk/datalog
 mkdir -p ~/data/zk/log
 mkdir -p ~/data/tmp/hadoop
 mkdir -p ~/data/journaldata
```



## 安装步骤

### 1.安装配置zookeeper集群(主要在node1上操作)

```shell
tar -C ~/app -xf zookeeper-3.4.9.tar.gz 
ln -s ~/app/zookeeper-3.4.9 ~/app/zookeeper
```

#### (1)配置参数文件（zoo.cfg)

```text
dataDir=/home/data/data/zk/data
dataLogDir=/home/data/data/zk/datalog
server.1=node1:2888:3888
server.2=node2:2888:3888
server.3=node3:2888:3888
```

#### (2)配置zk相关环境变量，并使其生效

```shell
export ZOOKEEPER_HOME="${HOME}/app/zookeeper"
export ZOO_LOG_DIR="${HOME}/data/zk/log"
export ZOO_LOG4J_PROP="INFO,ROLLINGFILE"
```

#### (3)同步zookeeper到node2,node3集群

```shell
 rsync -e "ssh" -az /home/data/app/zookeeper node2:/home/data/app
 rsync -e "ssh" -az /home/data/app/zookeeper node3:/home/data/app
 rsync -e "ssh" -az /home/data/app/zookeeper-3.4.9 node2:/home/data/app
 rsync -e "ssh" -az /home/data/app/zookeeper-3.4.9 node3:/home/data/app 
```

#### (4)配置zk的id(需要在zk集群上操作)

```shell
#node1
echo "1" >${HOME}/data/zk/data/myid
#node2
echo "2" >${HOME}/data/zk/data/myid
#node3
echo "3" >${HOME}/data/zk/data/myid
```

#### (5)启动zk集群(每台zk所在集群执行)

```shell
cd ${ZOOKEEPER_HOME} && ./bin/zkServer.sh start &
#检查zkServer的状态
cd ${ZOOKEEPER_HOME} && ./bin/zkServer.sh status &
#关闭zkServer
cd ${ZOOKEEPER_HOME} && ./bin/zkServer.sh stop &
```

### 2.安装配置Hadoop集群(主要在Node1上操作)



```shell
tar -C ~/app -xf hadoop-2.7.4.tar.gz 
ln -s ~/app/hadoop-2.7.4 ~/app/hadoop
```

####  (1）配置环境变量，并使其生效

 ```shell
export HADOOP_HOME="${HOME}/app/hadoop"
 ```

#### (2)修改配置文件

**修改core-site.xml文件**

```xml
<configuration>
    <!-- 指定hdfs的nameservice为ns1 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://ns1/</value>
    </property>
    <!-- 指定hadoop临时目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/data/data/tmp/hadoop</value>
    </property>

    <!-- 指定zookeeper地址 -->
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>node1:2181,node2:2181,node3:2181</value>
    </property>
</configuration>
```

**修改hdfs-site.xml文件**

```xml
  <!--指定hdfs的nameservice为ns1，需要和core-site.xml中的保持一致 -->
    <property>
        <name>dfs.nameservices</name>
        <value>ns1</value>
    </property>
    <!-- ns1下面有两个NameNode，分别是nn1，nn2 -->
    <property>
        <name>dfs.ha.namenodes.ns1</name>
        <value>nn1,nn2</value>
    </property>
    <!-- nn1的RPC通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.ns1.nn1</name>
        <value>node1:9000</value>
    </property>
    <!-- nn1的http通信地址 -->
    <property>
        <name>dfs.namenode.http-address.ns1.nn1</name>
        <value>node1:50070</value>
    </property>
    <!-- nn2的RPC通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.ns1.nn2</name>
        <value>node2:9000</value>
    </property>
    <!-- nn2的http通信地址 -->
    <property>
        <name>dfs.namenode.http-address.ns1.nn2</name>
        <value>node2:50070</value>
    </property>
    <!-- 指定NameNode的edits元数据在JournalNode上的存放位置 -->
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://node1:8485;node2:8485;node3:8485/ns1</value>
    </property>
    <!-- 指定JournalNode在本地磁盘存放数据的位置 -->
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/home/data/data/journaldata</value>
    </property>
    <!-- 开启NameNode失败自动切换 -->
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
    <!-- 配置失败自动切换实现方式 -->
    <property>
        <name>dfs.client.failover.proxy.provider.ns1</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
    <!-- 配置隔离机制方法，多个机制用换行分割，即每个机制暂用一行-->
    <property>
        <name>dfs.ha.fencing.methods</name>
        <value>
            sshfence
        </value>
    </property>
    <!-- 使用sshfence隔离机制时需要ssh免登陆 -->
    <property>
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/home/data/.ssh/id_rsa</value>
    </property>
    <!-- 配置sshfence隔离机制超时时间 -->
    <property>
        <name>dfs.ha.fencing.ssh.connect-timeout</name>
        <value>30000</value>
    </property>
 <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///home/data/data/dfs/namenode</value>
    </property>
    <property>
        <name>dfs.datanode.name.dir</name>
        <value>file:///home/data/data/dfs/datanode</value>
    </property>
</configuration>
```

**修改mapred-site.xml**

```XML
<configuration>
    <!-- 指定mr框架为yarn方式 -->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

**修改yarn-site.xm**

```xml
<configuration>
    <!-- 开启RM高可用 -->
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
    <!-- 指定RM的cluster id -->
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>yarncluster1</value>
    </property>
    <!-- 指定RM的名字 -->
    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>
    <!-- 分别指定RM的地址 -->
    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>node1</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>node2</value>
    </property>
    <!-- 指定zk集群地址 -->
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <value>node1:2181,node2:2181,node3:2181</value>
    </property>
    <property>
     <name>yarn.resourcemanager.webapp.address.rm1</name>
     <value>node1:8088</value>
     </property>
    <property>
     <name>yarn.resourcemanager.webapp.address.rm2</name>
     <value>node2:8088</value>
     </property>
     <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>1024</value>
    </property>
    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>1</value>
    </property>
</configuration>
```



**配置slaves**

```text
node1
node2
node3
```

**拷贝到其它节点**

```shell
rsync -e "ssh" -az /home/data/app/hadoop node2:/home/data/app
rsync -e "ssh" -az /home/data/app/hadoop node3:/home/data/app
rsync -e "ssh" -az /home/data/app/hadoop-2.7.4 node2:/home/data/app
rsync -e "ssh" -az /home/data/app/hadoop-2.7.4 node3:/home/data/app
```



**在每个journal节点(node1,node2,node3)上启动journalnode进程**

```shell
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh start journalnode &
```



**在namenode节点格式化**

```shell
#只在node1节点上执行
cd ${HADOOP_HOME} && bin/hdfs namenode -format 

```

**格式化ZKFC**

```shell
#只在node1上执行
cd ${HADOOP_HOME} && bin/hdfs namenode -format
cd ${HADOOP_HOME} && bin/hdfs zkfc -formatZK
```

**启动**

```shell
#只在node1上执行
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh start namenode &
#只在node2上执行
cd ${HADOOP_HOME} && bin/hdfs namenode -bootstrapStandby
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh start namenode &

#在node1,node2上执行
cd ${HADOOP_HOME} && sbin/yarn-daemon.sh start resourcemanager

#启动所有数据节点(node1,node2,node3)
cd ${HADOOP_HOME} && sbin/hadoop-daemons.sh start datanode
#在node1启动yarn
cd ${HADOOP_HOME} && sbin/start-yarn.sh
#在node1,node2启动zkfc
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh start zkfc
```

 **检查**

```shell
 #检查namenode的状态
cd ${HADOOP_HOME} &&  bin/hdfs haadmin -getServiceState nn1
cd ${HADOOP_HOME} && bin/hdfs haadmin -getServiceState nn2
 #检查rm的状态
cd ${HADOOP_HOME} &&  bin/yarn rmadmin -getServiceState rm1
cd ${HADOOP_HOME} && bin/yarn rmadmin -getServiceState rm2
```



## 关闭集群

**第一种方法**

```shell


#在node1,node2上关闭namenode
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh stop namenode
#在关闭datanode
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh stop datanode
#在node1,node2上关闭journalNode
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh stop journalnode

#在node1,node2上关闭zkfc
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh stop zkfc

#node1上关闭resouremanage
cd ${HADOOP_HOME} && sbin/yarn-daemon.sh stop resourcemanager
#node1上关闭nodemanager
cd ${HADOOP_HOME} && sbin/yarn-daemon.sh stop nodemanager

#终止zk集群（在node1,node2,node）
cd ${ZOOKEEPER_HOME} && bin/zkServer.sh stop
```

**第二种方法**

```shell
#在node1上执行
cd ${HADOOP_HOME} && sbin/stop-dfs.sh 
#在node1上执行
cd ${HADOOP_HOME} && sbin/stop-yarn.sh 
#在node2上执行
cd ${HADOOP_HOME} && sbin/yarn-daemon.sh stop resourcemanager
#在node1,node2,node3上执行
cd ${ZOOKEEPER_HOME} && bin/zkServer.sh stop
```



## 启动集群

**第一种方法**

```shell
#启动zkServer集群（node1,node2,node3)
cd ${ZOOKEEPER_HOME} && bin/zkServer.sh start
#启动NameNode(node1,node2)
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh start namenode
#启动node1,node2,node3上DataNode
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh start datanode
#启动Journalnode集群（node1,node2,node3)
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh start journalnode
#启动node1,node2上的zkfc
cd ${HADOOP_HOME} && sbin/hadoop-daemon.sh start zkfc

#node1,node2，node3上启动nodemanager
cd ${HADOOP_HOME} && sbin/yarn-daemon.sh start nodemanager
#node1，node2上启动resouremanage
cd ${HADOOP_HOME} && sbin/yarn-daemon.sh start resourcemanager

```

**第二种方法**

```shell
#在node1上执行
cd ${HADOOP_HOME} && sbin/start-dfs.sh 
#在node1上执行
cd ${HADOOP_HOME} && sbin/start-yarn.sh 
#在node2上执行
cd ${HADOOP_HOME} && sbin/yarn-daemon.sh start resourcemanager
```





## 问题登记##

1.部分节点的NodeManager无法启动

**异常日志**

```shell
2017-11-17 12:49:17,783 INFO org.apache.hadoop.service.AbstractService: Service NodeManager failed in state STARTED; cause: org.apache.hadoop.yarn.exceptions.YarnRuntimeException: org.apache.hadoop.yarn.exceptions.YarnRuntimeException: Recieved SHUTDOWN signal from Resourcemanager ,Registration of NodeManager failed, Message from ResourceManager: NodeManager from  node2 doesn't satisfy minimum allocations, Sending SHUTDOWN signal to the NodeManager.
org.apache.hadoop.yarn.exceptions.YarnRuntimeException: org.apache.hadoop.yarn.exceptions.YarnRuntimeException: Recieved SHUTDOWN signal from Resourcemanager ,Registration of NodeManager failed, Message from ResourceManager: NodeManager from  node2 doesn't satisfy minimum allocations, Sending SHUTDOWN signal to the NodeManager.
        at org.apache.hadoop.yarn.server.nodemanager.NodeStatusUpdaterImpl.serviceStart(NodeStatusUpdaterImpl.java:203)
        at org.apache.hadoop.service.AbstractService.start(AbstractService.java:193)
        at org.apache.hadoop.service.CompositeService.serviceStart(CompositeService.java:120)
        at org.apache.hadoop.yarn.server.nodemanager.NodeManager.serviceStart(NodeManager.java:272)
        at org.apache.hadoop.service.AbstractService.start(AbstractService.java:193)
        at org.apache.hadoop.yarn.server.nodemanager.NodeManager.initAndStartNodeManager(NodeManager.java:496)
        at org.apache.hadoop.yarn.server.nodemanager.NodeManager.main(NodeManager.java:543)
Caused by: org.apache.hadoop.yarn.exceptions.YarnRuntimeException: Recieved SHUTDOWN signal from Resourcemanager ,Registration of NodeManager failed, Message from ResourceManager: NodeManager from  node2 doesn't satisfy minimum allocations, Sending SHUTDOWN signal to the NodeManager.
        at org.apache.hadoop.yarn.server.nodemanager.NodeStatusUpdaterImpl.registerWithRM(NodeStatusUpdaterImpl.java:278)
        at org.apache.hadoop.yarn.server.nodemanager.NodeStatusUpdaterImpl.serviceStart(NodeStatusUpdaterImpl.java:197)
        ... 6 more
```

**解决办法：**

在yarn-site.xml调整或增加下列参数配置

```xml
<property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>1024</value>
    </property>
    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>1</value>
    </property>
```





## 官方参考命令



To start a Hadoop cluster you will need to start both the HDFS and YARN cluster.

The first time you bring up HDFS, it must be formatted. Format a new distributed filesystem as *hdfs*:

```
[hdfs]$ $HADOOP_PREFIX/bin/hdfs namenode -format <cluster_name>

```

Start the HDFS NameNode with the following command on the designated node as *hdfs*:

```
[hdfs]$ $HADOOP_PREFIX/sbin/hadoop-daemon.sh --config $HADOOP_CONF_DIR --script hdfs start namenode

```

Start a HDFS DataNode with the following command on each designated node as *hdfs*:

```
[hdfs]$ $HADOOP_PREFIX/sbin/hadoop-daemons.sh --config $HADOOP_CONF_DIR --script hdfs start datanode

```

If `etc/hadoop/slaves` and ssh trusted access is configured (see [Single Node Setup](http://hadoop.apache.org/docs/r2.7.4/hadoop-project-dist/hadoop-common/SingleCluster.html)), all of the HDFS processes can be started with a utility script. As *hdfs*:

```
[hdfs]$ $HADOOP_PREFIX/sbin/start-dfs.sh

```

Start the YARN with the following command, run on the designated ResourceManager as *yarn*:

```
[yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR start resourcemanager

```

Run a script to start a NodeManager on each designated host as *yarn*:

```
[yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemons.sh --config $HADOOP_CONF_DIR start nodemanager

```

Start a standalone WebAppProxy server. Run on the WebAppProxy server as *yarn*. If multiple servers are used with load balancing it should be run on each of them:

```
[yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR start proxyserver

```

If `etc/hadoop/slaves` and ssh trusted access is configured (see [Single Node Setup](http://hadoop.apache.org/docs/r2.7.4/hadoop-project-dist/hadoop-common/SingleCluster.html)), all of the YARN processes can be started with a utility script. As *yarn*:

```
[yarn]$ $HADOOP_PREFIX/sbin/start-yarn.sh

```

Start the MapReduce JobHistory Server with the following command, run on the designated server as *mapred*:

```
[mapred]$ $HADOOP_PREFIX/sbin/mr-jobhistory-daemon.sh --config $HADOOP_CONF_DIR start historyserver

```

### Hadoop Shutdown

Stop the NameNode with the following command, run on the designated NameNode as *hdfs*:

```
[hdfs]$ $HADOOP_PREFIX/sbin/hadoop-daemon.sh --config $HADOOP_CONF_DIR --script hdfs stop namenode

```

Run a script to stop a DataNode as *hdfs*:

```
[hdfs]$ $HADOOP_PREFIX/sbin/hadoop-daemons.sh --config $HADOOP_CONF_DIR --script hdfs stop datanode

```

If `etc/hadoop/slaves` and ssh trusted access is configured (see [Single Node Setup](http://hadoop.apache.org/docs/r2.7.4/hadoop-project-dist/hadoop-common/SingleCluster.html)), all of the HDFS processes may be stopped with a utility script. As *hdfs*:

```
[hdfs]$ $HADOOP_PREFIX/sbin/stop-dfs.sh

```

Stop the ResourceManager with the following command, run on the designated ResourceManager as *yarn*:

```
[yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR stop resourcemanager

```

Run a script to stop a NodeManager on a slave as *yarn*:

```
[yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemons.sh --config $HADOOP_CONF_DIR stop nodemanager

```

If `etc/hadoop/slaves` and ssh trusted access is configured (see [Single Node Setup](http://hadoop.apache.org/docs/r2.7.4/hadoop-project-dist/hadoop-common/SingleCluster.html)), all of the YARN processes can be stopped with a utility script. As *yarn*:

```
[yarn]$ $HADOOP_PREFIX/sbin/stop-yarn.sh

```

Stop the WebAppProxy server. Run on the WebAppProxy server as *yarn*. If multiple servers are used with load balancing it should be run on each of them:

```
[yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR stop proxyserver

```

Stop the MapReduce JobHistory Server with the following command, run on the designated server as *mapred*:

```
[mapred]$ $HADOOP_PREFIX/sbin/mr-jobhistory-daemon.sh --config $HADOOP_CONF_DIR stop historyserver
```