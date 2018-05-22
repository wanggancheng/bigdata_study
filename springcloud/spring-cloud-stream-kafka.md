#Spring Cloud Stream与Kafka的相关参数
##stream基本参数
*spring.cloud.stream.instanceIndex:The instance index of the application: a number from 0 to instanceCount-1
*spring.cloud.stream.instanceCount:
>     
Topic在逻辑上可以被认为是一个queue。每条消费都必须指定它的topic，可以简单理解为必须指明把这条消息放进哪个queue里。为了使得 Kafka的吞吐率可以水平扩展，物理上把topic分成一个或多个partition，每个partition在物理上对应一个文件夹，该文件夹下存储 这个partition的所有消息和索引文件。partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。
>
同一个partition内的消息只能被同一个组中的一个consumer消费。
>
当消费者数量多于partition的数量时，多余的消费者空闲。
>
消费者少于和等于partition的数量时，会出现多个partition对应一个消费者的情况，个别消费者消费量会比其他的多。


##bindings通用参数
* spring.cloud.stream.bindings.<channelName>.destination:目标
* sprinig.cloud.stream.bindings.<channelName>.group:分组名

##Kafka extend binding参数(KafkaExtendedBindingProperties.java)
配置的前缀：spring.cloud.stream.kafka
Kafka相关Configuration类在：META-INF/spring.binders中指定
kafka:\
org.springframework.cloud.stream.binder.kafka.config.KafkaBinderConfiguration

##kafka binder参数(KafkaBinderConfigurationProperties.java)
* spring.cloud.stream.kafka.binder.brokers:kafka　brokders参数，多个以逗号分隔
* spring.cloud.stream.kafka.binder.zk-nodes:zookeeper地址信息，多个以逗号分隔
* spring.cloud.stream.kafka.binder.autoAddPartitions:是否自动添加分区数
* spring.cloud.stream.kafka.binder.autoCreateTopics:是否自动创建Topic
* spring.cloud.stream.kafka.binder.offsetUpdateTimeWindow:
* spring.cloud.stream.kafka.binder.offsetUpdateCount:
* spring.cloud.stream.kafka.binder.maxWait:
* spring.cloud.stream.kafka.binder.socketBufferSize:
* spring.cloud.stream.kafka.binder.zkSessionTimeout:
* spring.cloud.stream.kafka.binder.zkConnectionTimeout:
* spring.cloud.stream.kafka.binder.requiredAcks:
* spring.cloud.stream.kafka.binder.fetchSize:
* spring.cloud.stream.kafka.binder.queueSize:


## producer参数
* spring.cloud.stream.bindings.<channelName>.producer.partitionCount:
* spring.cloud.stream.bindings.<channelName>.producer.headerMode:
* spring.cloud.stream.bindings.<channelName>.producer.partitionKeyExtractorClass
* spring.cloud.stream.bindings.<channelName>.producer.partitionSelectorClass

## kafka producer参数
* spring.cloud.stream.kafka.bindings.<channelName>.producer.bufferSize:
* spring.cloud.stream.kafka.bindings.<channelName>.producer.maxRequestSize:
* spring.cloud.stream.kafka.bindings.<channelName>.producer.sync:
* spring.cloud.stream.kafka.bindings.<channelName>.producer.batchTimeout:



##consumer参数
* spring.cloud.stream.bindings.<channelName>.consumer.autoCommitOffset:是否自动提交Offset
* spring.cloud.stream.bindings.<channelName>.consumer.concurrency:并发数
* spring.cloud.stream.bindings.<channelName>.consumer.partitioned:是否分区
##Kafka 扩展consumer参数
* spring.cloud.stream.kafka.bindings.<channelName>.consumer.resetOffsets:是否重置offset
* spring.cloud.stream.kafka.bindings.<channelName>.consumer.startOffset:earliest
* spring.cloud.stream.kafka.bindings.<channelName>.consumer.enableDlq:
* spring.cloud.stream.kafka.bindings.<channelName>.consumer.recoveryInterval:恢复间隔

