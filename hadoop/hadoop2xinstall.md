## 搭建hadoop集群环境

### 系统环境准备

**创建用户组及用户**

```sh
groupadd -g 10000 bigdata
useradd -m -g bigdata -u 10000 bigdata
passwd bigdata
```

**安装jdk7**

```sh
tar -C /opt/java -xf jdk-7u80-linux-x64
```

**配置Java环境变量\(/home/bigdata/.bashrc\)**

```sh
JAVA_HOME=/opt/java/jdk1.7.0_80
PATH=${JAVA_HOME}/bin:$PATH
CLASSPATH=.:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export JAVA_HOME
export PATH
export CLASSPATH
HADOOP_HOME=/opt/hadoop/hadoop-2.6.0
export HADOOP_HOME
```

```sh
#创建/opt/hadoop目录，所有者为bigdata
sudo mkdir -p /opt/hadoop
sudo chown -R bigdata:bigdata /opt/hadoop
```

**配置ssh无密码登录**

1. 设置hosts文件。在/etc/hosts/文件中配置IP与HOSTNAME的映射

```sh
192.168.56.101  centos71 master
192.168.56.102  centos72 slave1
192.168.56.103  centos73 slave2
192.168.56.104  centos74 slave3
```

1. 生成公钥和私钥，执行ssh-keygen -t rsa,接着连续按3次回车键

```sh
ssh-keygen  -t rsa
```

​    接着分别执行：

```sh
ssh-copy-id -i /home/bigdata/.ssh/id_rsa.pub bigdata@master
ssh-copy-id -i /home/bigdata/.ssh/id_rsa.pub bigdata@slave1
ssh-copy-id -i /home/bigdata/.ssh/id_rsa.pub bigdata@slave2
ssh-copy-id -i /home/bigdata/.ssh/id_rsa.pub bigdata@slave3
```

​    接着验证无密码登录是否配置成功：

```sh
ssh master
ssh slave1
ssh slave2
ssh slave3
```

**安装配置NTP**

```sh
#安装ntp，配置开机自启动，当前启动ntpd
sudo yum install -y ntp
sudo systemctl enable ntpd
sudo systemctl start ntpd
sudo ntpdate -u cn.pool.ntp.org
```

​    配置/etc/ntp.conf

下面在master上配置（当作ntp server服务器）

```sh
#在#restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap后增加下行
restrict 192.168.56.0 mask 255.255.255.0 nomodify notrap
#注释server 0.centos.pool.ntp.org iburst\nserver 1.centos.pool.ntp.org iburst\nserver 2.centos.pool.ntp.org iburst \nserver 3.centos.pool.ntp.org iburst
#增加下列三行
server 2.cn.pool.ntp.org
server 1.asia.pool.ntp.org
server 2.asia.pool.ntp.org
server 127.127.1.0  # local clock
fudge 127.127.1.0  stratum 10
#在# Enable public key cryptography.
restrict 2.cn.pool.ntp.org nomodify notrap noquery
restrict 1.asia.pool.ntp.org nomodify notrap noquery
restrict 2.asia.pool.ntp.org nomodify notrap noquery
```

下面在slave1,slave2,slave3上配置

```sh
#把# Please consider joining the pool 下面server开头的几行注释掉
#增加master作为ntp本地服务器
#配置上游时间服务器为本地的ntpd Server服务器
server master
# 配置允许上游时间服务器主动修改本机的时间
server master
server 127.127.1.0
fudge 127.127.1.0  stratum 10
```

### 配置Hadoop集群

上转Hadoop安装包到master机器，并解压到/opt/hadoop/

涉及的配置文件有：

```sh
${HADOOP_HOME}/etc/hadoop/hadoop-env.sh
${HADOOP_HOME}/etc/hadoop/yarn-env.sh
${HADOOP_HOME}/etc/hadoop/slaves
${HADOOP_HOME}/etc/hadoop/core-site.xml
${HADOOP_HOME}/etc/hadoop/hdfs-site.xml
${HADOOP_HOME}/etc/hadoop/mapred-site.xml
${HADOOP_HOME}/etc/hadoop/yarn-site.xml
```

1\)配置文件1:hadoop-env.sh

```sh
export JAVA_HOME=${JAVA_HOME} #如果用户的环境变量未设置或版本不对，可调整此参数
```

2）配置文件2：yarn-env.sh

```sh
#如果用户的环境变量未设置或版本不对，可调整此参数
```

3）配置文件3:slaves

```sh
slave1
slave2
slave3
```

4\)配置文件4：core-site.xml

需先执行下列命令：

sudo mkdir -p /var/log/hadoop/tmp

sudo chown -R bigdata:bigdata /var/log/hadoop

```sh
<configuration>
   <property>
             <name>fs.defaultFS</name>
             <value>hdfs://master:8020</value>
  </property>
  <property>
            <name>hadoop.tmp.dir</name>
            <value>/var/log/hadoop/tmp</value>
  </property>
</configuration>
```

5\)配置文件5：hdfs-site.xml

需先执行下列命令：

sudo mkdir -p /data/hadoop/hdfs/name

sudo mkdir  -p  /data/hadoop/hdfs/data

sudo chown -R bigdata:bigdata /data

```sh
<configuration>
    <property>
     <name>dfs.namenode.name.dir</name>
     <value>file:///data/hadoop/hdfs/name</value>
   </property>
   <property>
        <name>dfs.datanode.data.dir</name>
       <value>file:///data/hadoop/hdfs/data</value>
   </property>
   <property>
              <name>dfs.namenode.secondary.http-address</name>
              <value>master:50090</value>
  </property>
   <property>
          <name>dfs.replication</name>
          <value>3</value>
  </property>

</configuration>
```

6\)配置文件6：mapred-site.xml

```sh
cp mapred-site.xml.template  mapred-site.xml
```

```sh
<configuration>
   <property>
   <name>mapreduce.framework.name</name>
   <value>yarn</value>
  </property>
  <property>
          <name>mapreduce.jobhistory.address</name>
          <value>master:10020</value>
  </property>
  <property>
           <name>mapreduce.jobhistory.webapp.address</name>
           <value>master:19888</value>
  </property>

</configuration>
```

7\)配置文件7:yarn-site.xml



```xml
<configuration>
  <property>
    <description>The hostname of the RM.</description>
    <name>yarn.resourcemanager.hostname</name>
    <value>master</value>
  </property>
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>${yarn.resourcemanager.hostname}:8032</value>
  </property>
  <property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>${yarn.resourcemanager.hostname}:8030</value>
  </property>
  <property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>${yarn.resourcemanager.hostname}:8088</value>
  </property>
  <property>
    <name>yarn.resourcemanager.webapp.https.address</name>
    <value>${yarn.resourcemanager.hostname}:8090</value>
  </property>
  <property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>${yarn.resourcemanager.hostname}:8031</value>
  </property>
  <property>
    <name>yarn.resourcemanager.admin.address</name>
    <value>${yarn.resourcemanager.hostname}:8033</value>
  </property>
  <property>
    <name>yarn.nodemanager.local-dirs</name>
    <value>/data/hadoop/yarn/local</value>
  </property>
  <property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
  </property>
   <property>
    <name>yarn.nodemanager.remote-app-log-dir</name>
    <value>/data/tmp/logs</value>
  </property>
  <property>
  <name>yarn.log.server.url</name>
  <value>http://master:19888/jobhistory/logs/</value>
 </property>
 <property>
  <name>yarn.nodemanager.vmem-check-enabled</name>
 <value>false</value>
 </property>
 <property>
 <name>yarn.nodemanager.aux-services</name>
 <value>mapreduce_shuffle</value>
 </property>
<property>
<name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
<value>org.apache.hadoop.mapred.ShuffleHandler</value>
</property>
</configuration>
```

### 复制程序到其它节点

```sh
 scp -r /opt/hadoop/hadoop-2.6.0/ slave1:/opt/hadoop
 scp -r /opt/hadoop/hadoop-2.6.0/ slave2:/opt/hadoop
 scp -r /opt/hadoop/hadoop-2.6.0/ slave3:/opt/hadoop
```

### 格式化NameNode

```sh
${HADOOP_HOME}/bin/hdfs namenode -format
```

### 集群启动关闭与监控

​    启动集群，只要在master节点（NameNode服务所在节点）执行下列命令

**启动集群**

```sh
cd $HADOOP_HOME
sbin/start-dfs.sh
sbin/start-yarn.sh
sbin/mr-jobhistory-daemon.sh start historyserver
```

**关闭集群**

```sh
cd $HADOOP_HOME
sbin/stop-yarn.sh
sbin/stop-dfs.sh
sbin/mr-jobhistory-daemon.sh stop historyserver
```

**查看集群状态**

```sh
bin/hdfs dfsadmin -report
```

**hadoop集群监控相关端口**

| 服务 | Web接口 | 默认端口 |
| --- | --- | --- |
| NameNode | [http://namenode\_host:port/](http://namenode_host:port/) | 50070 |
| ResourceManager | [http://resourcemanager\_host:port/](http://resourcemanager_host:port/) | 8088 |
| MapReduce JobHistory Server | [http://jobhistoryserver\_host:port/](http://jobhistoryserver_host:port/) | 19888 |



