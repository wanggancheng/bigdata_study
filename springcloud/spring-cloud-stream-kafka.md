#Spring Cloud Stream与Kafka的相关参数
##stream基本参数
*spring.cloud.stream.instanceIndex:The instance index of the application: a number from 0 to instanceCount-1
*spring.cloud.stream.instanceCount:



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
