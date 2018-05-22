#Spring Cloud Stream与Kafka的相关参数
##stream基本参数
*spring.cloud.stream.instanceIndex:The instance index of the application: a number from 0 to instanceCount-1
*spring.cloud.stream.instanceCount:
Topic在逻辑上可以被认为是一个queue。每条消费都必须指定它的topic，可以简单理解为必须指明把这条消息放进哪个queue里。为了使得 Kafka的吞吐率可以水平扩展，物理上把topic分成一个或多个partition，每个partition在物理上对应一个文件夹，该文件夹下存储 这个partition的所有消息和索引文件。partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。

同一个partition内的消息只能被同一个组中的一个consumer消费。

当消费者数量多于partition的数量时，多余的消费者空闲。

消费者少于和等于partition的数量时，会出现多个partition对应一个消费者的情况，个别消费者消费量会比其他的多。

##bindings通用参数
* spring.cloud.stream.bindings.<bindingName>.destination:目标
* sprinig.cloud.stream.bindings.<bindingName>.group:分组名


##kafka binder参数
* spring.cloud.stream.kafka.binder.brokers:kafka　brokders参数，多个以逗号分隔
* spring.cloud.stream.kafka.binder.zk-nodes:zookeeper地址信息，多个以逗号分隔
* spring.cloud.stream.kafka.binder.autoAddPartitions:是否自动添加分区数
* spring.cloud.stream.kafka.binder.autoCreateTopics:是否自动创建Topic
* spring.cloud.stream.kafka.binder.minPartitionCount:最少分区数

##Kafka producer参数

##Kafka consumer参数
* spring.cloud.stream.bindings.<bindingName>.consumer.autoCommitOffset:是否自动提交Offset
* spring.cloud.stream.bindings.<bindingName>.consumer.concurrency:并发数
* spring.cloud.stream.bindings.<bindingName>.consumer.partitioned:是否分区
