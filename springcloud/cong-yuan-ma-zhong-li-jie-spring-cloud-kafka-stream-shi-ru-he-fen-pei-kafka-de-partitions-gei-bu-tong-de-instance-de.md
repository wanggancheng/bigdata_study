内容转自:https://blog.csdn.net/xiao_jun_0820/article/details/80166131

首先，kafka的topic是由多个partitions物理分隔的。假设topic: testIn,有8个partitions

其次，我们编写的springcloud kafka stream程序，打成jar包后，可以部署多个不同的实例instances，假设部署了3个instance。

那么这3个instance是怎么分配这8个partitions的呢？

在spring.cloud.stream.kafka.bindings.input.consumer.autoRebalanceEnabled=true时(默认)，多个实例会自动均衡

的分配partitions,由ConsumerCoordinator来自动协调，不用您操心了（跟instanceCount 和instanceIndex 没关系了，但是concurrency还是同理的）， 当设置成false后：

主要是通过consumer的4个参数(org.springframework.cloud.stream.binder.ConsumerProperties)来决定的:



spring.cloud.stream.bindings.input.consumer.partitioned=true

准备启动多少个实例

spring.cloud.stream.bindings.input.consumer.instanceCount =3

该实例编号index,从0开始到instanceCount -1

spring.cloud.stream.bindings.input.consumer.instanceIndex =0

每个实例中启动多少个kafka consumer

spring.cloud.stream.bindings.input.consumer.concurrency  =1



如果按照上面的配置的话(3个instance,每个instance的并行度1):

instance0将启动一个consumer0,消费p0,p3,p6; 

instance1将启动一个consumer0,消费p1,p4,p7; 

instance2将启动一个consumer0,消费p2,p5;

主要逻辑在org.springframework.cloud.stream.binder.kafka.KafkaMessageChannelBinder#createConsumerEndpoint这个方法中:

[java] view plain copy
protected MessageProducer createConsumerEndpoint(final ConsumerDestination destination, final String group,  
            final ExtendedConsumerProperties<KafkaConsumerProperties> extendedConsumerProperties) {  
  
        boolean anonymous = !StringUtils.hasText(group);  
        Assert.isTrue(!anonymous || !extendedConsumerProperties.getExtension().isEnableDlq(),  
                "DLQ support is not available for anonymous subscriptions");  
        String consumerGroup = anonymous ? "anonymous." + UUID.randomUUID().toString() : group;  
        final ConsumerFactory<?, ?> consumerFactory = createKafkaConsumerFactory(anonymous, consumerGroup,  
                extendedConsumerProperties);  
        int partitionCount = extendedConsumerProperties.getInstanceCount()  
                * extendedConsumerProperties.getConcurrency();  
        //如果AutoRebalanceEnabled=false的话，当instanceCount*Concurrency比实际topic的partitions数量还要多的话，将报错，因为会启动空闲的consumer  
        Collection<PartitionInfo> allPartitions = provisioningProvider.getPartitionsForTopic(partitionCount,  
                extendedConsumerProperties.getExtension().isAutoRebalanceEnabled(),  
                new Callable<Collection<PartitionInfo>>() {  
  
                    @Override  
                    public Collection<PartitionInfo> call() throws Exception {  
                        Consumer<?, ?> consumer = consumerFactory.createConsumer();  
                        List<PartitionInfo> partitionsFor = consumer.partitionsFor(destination.getName());  
                        consumer.close();  
                        return partitionsFor;  
                    }  
  
                });  
  
        Collection<PartitionInfo> listenedPartitions;  
  
        if (extendedConsumerProperties.getExtension().isAutoRebalanceEnabled() ||  
                extendedConsumerProperties.getInstanceCount() == 1) {  
            listenedPartitions = allPartitions;  
        }  
        else {  
            listenedPartitions = new ArrayList<>();  
            //为该Instance计算应该分派哪些partitions  
            for (PartitionInfo partition : allPartitions) {  
                // divide partitions across modules  
                if ((partition.partition()  
                        % extendedConsumerProperties.getInstanceCount()) == extendedConsumerProperties  
                                .getInstanceIndex()) {  
                    listenedPartitions.add(partition);  
                }  
            }  
        }  
        this.topicsInUse.put(destination.getName(), new TopicInformation(group, listenedPartitions));  
  
        Assert.isTrue(!CollectionUtils.isEmpty(listenedPartitions), "A list of partitions must be provided");  
        final TopicPartitionInitialOffset[] topicPartitionInitialOffsets = getTopicPartitionInitialOffsets(  
                listenedPartitions);  
        final ContainerProperties containerProperties = anonymous  
                || extendedConsumerProperties.getExtension().isAutoRebalanceEnabled()  
                        ? new ContainerProperties(destination.getName())  
                        : new ContainerProperties(topicPartitionInitialOffsets);  
        //并行度取consumer参数中设置的Concurrency和实际处理的partitions子集中较小的一个。  
        int concurrency = Math.min(extendedConsumerProperties.getConcurrency(), listenedPartitions.size());  
        @SuppressWarnings("rawtypes")  
        //ConcurrentMessageListenerContainer的doStart方法将根据并行度创建consumer  
        final ConcurrentMessageListenerContainer<?, ?> messageListenerContainer =  
                new ConcurrentMessageListenerContainer(  
                        consumerFactory, containerProperties) {  
  
            @Override  
            public void stop(Runnable callback) {  
                super.stop(callback);  
            }  
  
        };  
        messageListenerContainer.setConcurrency(concurrency);  
        if (!extendedConsumerProperties.getExtension().isAutoCommitOffset()) {  
            messageListenerContainer.getContainerProperties()  
                    .setAckMode(AbstractMessageListenerContainer.AckMode.MANUAL);  
            messageListenerContainer.getContainerProperties().setAckOnError(false);  
        }  
        else {  
            messageListenerContainer.getContainerProperties()  
                    .setAckOnError(isAutoCommitOnError(extendedConsumerProperties));  
        }  
        if (this.logger.isDebugEnabled()) {  
            this.logger.debug(  
                    "Listened partitions: " + StringUtils.collectionToCommaDelimitedString(listenedPartitions));  
        }  
        final KafkaMessageDrivenChannelAdapter<?, ?> kafkaMessageDrivenChannelAdapter = new KafkaMessageDrivenChannelAdapter<>(  
                messageListenerContainer);  
        kafkaMessageDrivenChannelAdapter.setBeanFactory(this.getBeanFactory());  
        ErrorInfrastructure errorInfrastructure = registerErrorInfrastructure(destination, consumerGroup,  
                extendedConsumerProperties);  
        if (extendedConsumerProperties.getMaxAttempts() > 1) {  
            kafkaMessageDrivenChannelAdapter.setRetryTemplate(buildRetryTemplate(extendedConsumerProperties));  
            kafkaMessageDrivenChannelAdapter.setRecoveryCallback(errorInfrastructure.getRecoverer());  
        }  
        else {  
            kafkaMessageDrivenChannelAdapter.setErrorChannel(errorInfrastructure.getErrorChannel());  
        }  
        return kafkaMessageDrivenChannelAdapter;  
    }  
org.springframework.kafka.listener.ConcurrentMessageListenerContainer#doStart方法：


[java] view plain copy
@Override  
    protected void doStart() {  
        if (!isRunning()) {  
            ContainerProperties containerProperties = getContainerProperties();  
            TopicPartitionInitialOffset[] topicPartitions = containerProperties.getTopicPartitions();  
            if (topicPartitions != null  
                    && this.concurrency > topicPartitions.length) {  
                this.logger.warn("When specific partitions are provided, the concurrency must be less than or "  
                        + "equal to the number of partitions; reduced from " + this.concurrency + " to "  
                        + topicPartitions.length);  
                this.concurrency = topicPartitions.length;  
            }  
            setRunning(true);  
                        //根据并行度生成N个KafkaMessageListenerContainer，  
                        //每个KafkaMessageListenerContainer的构造函数中会初始化一个KafkaConsumer  
            for (int i = 0; i < this.concurrency; i++) {  
                KafkaMessageListenerContainer<K, V> container;  
                if (topicPartitions == null) {  
                    container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties);  
                }  
                else {  
                //从前面分给Instance的partitions子集中，再指派一些partitions给具体的并行consumer  
                    container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties,  
                            partitionSubset(containerProperties, i));  
                }  
                if (getBeanName() != null) {  
                    container.setBeanName(getBeanName() + "-" + i);  
                }  
                if (getApplicationEventPublisher() != null) {  
                    container.setApplicationEventPublisher(getApplicationEventPublisher());  
                }  
                container.setClientIdSuffix("-" + i);  
                container.start();  
                this.containers.add(container);  
            }  
        }  
    }  
  
  //具体的二级子集生成策略  
    private TopicPartitionInitialOffset[] partitionSubset(ContainerProperties containerProperties, int i) {  
        TopicPartitionInitialOffset[] topicPartitions = containerProperties.getTopicPartitions();  
        if (this.concurrency == 1) {  
            return topicPartitions;  
        }  
        else {  
            int numPartitions = topicPartitions.length;  
            if (numPartitions == this.concurrency) {  
                return new TopicPartitionInitialOffset[] { topicPartitions[i] };  
            }  
            else {  
                int perContainer = numPartitions / this.concurrency;  
                TopicPartitionInitialOffset[] subset;  
                if (i == this.concurrency - 1) {  
                    subset = Arrays.copyOfRange(topicPartitions, i * perContainer, topicPartitions.length);  
                }  
                else {  
                    subset = Arrays.copyOfRange(topicPartitions, i * perContainer, (i + 1) * perContainer);  
                }  
                return subset;  
            }  
        }  
    }  
看完以上核心代码，应该就很清楚partitions的分配策略了，下面我们改一下场景，启动2个instance,每个instance的concurrency=3，那么：

instance0将分到[p0,p2,p4,p6]这4个分区，然后会启动3个consumer,consumer0将消费[p0],consumer1消费[p2],consumer2消费[p4,p6]。

instance0将分到[p1,p3,p5,p7]这4个分区，然后会启动3个consumer,consumer0将消费[p1],consumer1消费[p3],consumer2消费[p5,p7]。

可以看到每个instance的consumer2其实会多消费一个分区，不是很均匀，所以应该尽可能能整除，比如启动2个instance，concurrency=4，或者concurrency=2。当concurrency=2时，每个实例的每个consumer将消费两个partition。

看到这里，你应该对这几个参数有了一个比较深刻的理解了，这样在stream程序扩容或者缩容的时候应该能正确的配置参数.首先，kafka的topic是由多个partitions物理分隔的。假设topic: testIn,有8个partitions

其次，我们编写的springcloud kafka stream程序，打成jar包后，可以部署多个不同的实例instances，假设部署了3个instance。

那么这3个instance是怎么分配这8个partitions的呢？

在spring.cloud.stream.kafka.bindings.input.consumer.autoRebalanceEnabled=true时(默认)，多个实例会自动均衡

的分配partitions,由ConsumerCoordinator来自动协调，不用您操心了（跟instanceCount 和instanceIndex 没关系了，但是concurrency还是同理的）， 当设置成false后：

主要是通过consumer的4个参数(org.springframework.cloud.stream.binder.ConsumerProperties)来决定的:



spring.cloud.stream.bindings.input.consumer.partitioned=true

准备启动多少个实例

spring.cloud.stream.bindings.input.consumer.instanceCount =3

该实例编号index,从0开始到instanceCount -1

spring.cloud.stream.bindings.input.consumer.instanceIndex =0

每个实例中启动多少个kafka consumer

spring.cloud.stream.bindings.input.consumer.concurrency  =1



如果按照上面的配置的话(3个instance,每个instance的并行度1):

instance0将启动一个consumer0,消费p0,p3,p6; 

instance1将启动一个consumer0,消费p1,p4,p7; 

instance2将启动一个consumer0,消费p2,p5;

主要逻辑在org.springframework.cloud.stream.binder.kafka.KafkaMessageChannelBinder#createConsumerEndpoint这个方法中:

[java] view plain copy
protected MessageProducer createConsumerEndpoint(final ConsumerDestination destination, final String group,  
            final ExtendedConsumerProperties<KafkaConsumerProperties> extendedConsumerProperties) {  
  
        boolean anonymous = !StringUtils.hasText(group);  
        Assert.isTrue(!anonymous || !extendedConsumerProperties.getExtension().isEnableDlq(),  
                "DLQ support is not available for anonymous subscriptions");  
        String consumerGroup = anonymous ? "anonymous." + UUID.randomUUID().toString() : group;  
        final ConsumerFactory<?, ?> consumerFactory = createKafkaConsumerFactory(anonymous, consumerGroup,  
                extendedConsumerProperties);  
        int partitionCount = extendedConsumerProperties.getInstanceCount()  
                * extendedConsumerProperties.getConcurrency();  
        //如果AutoRebalanceEnabled=false的话，当instanceCount*Concurrency比实际topic的partitions数量还要多的话，将报错，因为会启动空闲的consumer  
        Collection<PartitionInfo> allPartitions = provisioningProvider.getPartitionsForTopic(partitionCount,  
                extendedConsumerProperties.getExtension().isAutoRebalanceEnabled(),  
                new Callable<Collection<PartitionInfo>>() {  
  
                    @Override  
                    public Collection<PartitionInfo> call() throws Exception {  
                        Consumer<?, ?> consumer = consumerFactory.createConsumer();  
                        List<PartitionInfo> partitionsFor = consumer.partitionsFor(destination.getName());  
                        consumer.close();  
                        return partitionsFor;  
                    }  
  
                });  
  
        Collection<PartitionInfo> listenedPartitions;  
  
        if (extendedConsumerProperties.getExtension().isAutoRebalanceEnabled() ||  
                extendedConsumerProperties.getInstanceCount() == 1) {  
            listenedPartitions = allPartitions;  
        }  
        else {  
            listenedPartitions = new ArrayList<>();  
            //为该Instance计算应该分派哪些partitions  
            for (PartitionInfo partition : allPartitions) {  
                // divide partitions across modules  
                if ((partition.partition()  
                        % extendedConsumerProperties.getInstanceCount()) == extendedConsumerProperties  
                                .getInstanceIndex()) {  
                    listenedPartitions.add(partition);  
                }  
            }  
        }  
        this.topicsInUse.put(destination.getName(), new TopicInformation(group, listenedPartitions));  
  
        Assert.isTrue(!CollectionUtils.isEmpty(listenedPartitions), "A list of partitions must be provided");  
        final TopicPartitionInitialOffset[] topicPartitionInitialOffsets = getTopicPartitionInitialOffsets(  
                listenedPartitions);  
        final ContainerProperties containerProperties = anonymous  
                || extendedConsumerProperties.getExtension().isAutoRebalanceEnabled()  
                        ? new ContainerProperties(destination.getName())  
                        : new ContainerProperties(topicPartitionInitialOffsets);  
        //并行度取consumer参数中设置的Concurrency和实际处理的partitions子集中较小的一个。  
        int concurrency = Math.min(extendedConsumerProperties.getConcurrency(), listenedPartitions.size());  
        @SuppressWarnings("rawtypes")  
        //ConcurrentMessageListenerContainer的doStart方法将根据并行度创建consumer  
        final ConcurrentMessageListenerContainer<?, ?> messageListenerContainer =  
                new ConcurrentMessageListenerContainer(  
                        consumerFactory, containerProperties) {  
  
            @Override  
            public void stop(Runnable callback) {  
                super.stop(callback);  
            }  
  
        };  
        messageListenerContainer.setConcurrency(concurrency);  
        if (!extendedConsumerProperties.getExtension().isAutoCommitOffset()) {  
            messageListenerContainer.getContainerProperties()  
                    .setAckMode(AbstractMessageListenerContainer.AckMode.MANUAL);  
            messageListenerContainer.getContainerProperties().setAckOnError(false);  
        }  
        else {  
            messageListenerContainer.getContainerProperties()  
                    .setAckOnError(isAutoCommitOnError(extendedConsumerProperties));  
        }  
        if (this.logger.isDebugEnabled()) {  
            this.logger.debug(  
                    "Listened partitions: " + StringUtils.collectionToCommaDelimitedString(listenedPartitions));  
        }  
        final KafkaMessageDrivenChannelAdapter<?, ?> kafkaMessageDrivenChannelAdapter = new KafkaMessageDrivenChannelAdapter<>(  
                messageListenerContainer);  
        kafkaMessageDrivenChannelAdapter.setBeanFactory(this.getBeanFactory());  
        ErrorInfrastructure errorInfrastructure = registerErrorInfrastructure(destination, consumerGroup,  
                extendedConsumerProperties);  
        if (extendedConsumerProperties.getMaxAttempts() > 1) {  
            kafkaMessageDrivenChannelAdapter.setRetryTemplate(buildRetryTemplate(extendedConsumerProperties));  
            kafkaMessageDrivenChannelAdapter.setRecoveryCallback(errorInfrastructure.getRecoverer());  
        }  
        else {  
            kafkaMessageDrivenChannelAdapter.setErrorChannel(errorInfrastructure.getErrorChannel());  
        }  
        return kafkaMessageDrivenChannelAdapter;  
    }  
org.springframework.kafka.listener.ConcurrentMessageListenerContainer#doStart方法：


[java] view plain copy
@Override  
    protected void doStart() {  
        if (!isRunning()) {  
            ContainerProperties containerProperties = getContainerProperties();  
            TopicPartitionInitialOffset[] topicPartitions = containerProperties.getTopicPartitions();  
            if (topicPartitions != null  
                    && this.concurrency > topicPartitions.length) {  
                this.logger.warn("When specific partitions are provided, the concurrency must be less than or "  
                        + "equal to the number of partitions; reduced from " + this.concurrency + " to "  
                        + topicPartitions.length);  
                this.concurrency = topicPartitions.length;  
            }  
            setRunning(true);  
                        //根据并行度生成N个KafkaMessageListenerContainer，  
                        //每个KafkaMessageListenerContainer的构造函数中会初始化一个KafkaConsumer  
            for (int i = 0; i < this.concurrency; i++) {  
                KafkaMessageListenerContainer<K, V> container;  
                if (topicPartitions == null) {  
                    container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties);  
                }  
                else {  
                //从前面分给Instance的partitions子集中，再指派一些partitions给具体的并行consumer  
                    container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties,  
                            partitionSubset(containerProperties, i));  
                }  
                if (getBeanName() != null) {  
                    container.setBeanName(getBeanName() + "-" + i);  
                }  
                if (getApplicationEventPublisher() != null) {  
                    container.setApplicationEventPublisher(getApplicationEventPublisher());  
                }  
                container.setClientIdSuffix("-" + i);  
                container.start();  
                this.containers.add(container);  
            }  
        }  
    }  
  
  //具体的二级子集生成策略  
    private TopicPartitionInitialOffset[] partitionSubset(ContainerProperties containerProperties, int i) {  
        TopicPartitionInitialOffset[] topicPartitions = containerProperties.getTopicPartitions();  
        if (this.concurrency == 1) {  
            return topicPartitions;  
        }  
        else {  
            int numPartitions = topicPartitions.length;  
            if (numPartitions == this.concurrency) {  
                return new TopicPartitionInitialOffset[] { topicPartitions[i] };  
            }  
            else {  
                int perContainer = numPartitions / this.concurrency;  
                TopicPartitionInitialOffset[] subset;  
                if (i == this.concurrency - 1) {  
                    subset = Arrays.copyOfRange(topicPartitions, i * perContainer, topicPartitions.length);  
                }  
                else {  
                    subset = Arrays.copyOfRange(topicPartitions, i * perContainer, (i + 1) * perContainer);  
                }  
                return subset;  
            }  
        }  
    }  
看完以上核心代码，应该就很清楚partitions的分配策略了，下面我们改一下场景，启动2个instance,每个instance的concurrency=3，那么：

instance0将分到[p0,p2,p4,p6]这4个分区，然后会启动3个consumer,consumer0将消费[p0],consumer1消费[p2],consumer2消费[p4,p6]。

instance0将分到[p1,p3,p5,p7]这4个分区，然后会启动3个consumer,consumer0将消费[p1],consumer1消费[p3],consumer2消费[p5,p7]。

可以看到每个instance的consumer2其实会多消费一个分区，不是很均匀，所以应该尽可能能整除，比如启动2个instance，concurrency=4，或者concurrency=2。当concurrency=2时，每个实例的每个consumer将消费两个partition。

看到这里，你应该对这几个参数有了一个比较深刻的理解了，这样在stream程序扩容或者缩容的时候应该能正确的配置参数.首先，kafka的topic是由多个partitions物理分隔的。假设topic: testIn,有8个partitions

其次，我们编写的springcloud kafka stream程序，打成jar包后，可以部署多个不同的实例instances，假设部署了3个instance。

那么这3个instance是怎么分配这8个partitions的呢？

在spring.cloud.stream.kafka.bindings.input.consumer.autoRebalanceEnabled=true时(默认)，多个实例会自动均衡

的分配partitions,由ConsumerCoordinator来自动协调，不用您操心了（跟instanceCount 和instanceIndex 没关系了，但是concurrency还是同理的）， 当设置成false后：

主要是通过consumer的4个参数(org.springframework.cloud.stream.binder.ConsumerProperties)来决定的:



spring.cloud.stream.bindings.input.consumer.partitioned=true

准备启动多少个实例

spring.cloud.stream.bindings.input.consumer.instanceCount =3

该实例编号index,从0开始到instanceCount -1

spring.cloud.stream.bindings.input.consumer.instanceIndex =0

每个实例中启动多少个kafka consumer

spring.cloud.stream.bindings.input.consumer.concurrency  =1



如果按照上面的配置的话(3个instance,每个instance的并行度1):

instance0将启动一个consumer0,消费p0,p3,p6; 

instance1将启动一个consumer0,消费p1,p4,p7; 

instance2将启动一个consumer0,消费p2,p5;

主要逻辑在org.springframework.cloud.stream.binder.kafka.KafkaMessageChannelBinder#createConsumerEndpoint这个方法中:

[java] view plain copy
protected MessageProducer createConsumerEndpoint(final ConsumerDestination destination, final String group,  
            final ExtendedConsumerProperties<KafkaConsumerProperties> extendedConsumerProperties) {  
  
        boolean anonymous = !StringUtils.hasText(group);  
        Assert.isTrue(!anonymous || !extendedConsumerProperties.getExtension().isEnableDlq(),  
                "DLQ support is not available for anonymous subscriptions");  
        String consumerGroup = anonymous ? "anonymous." + UUID.randomUUID().toString() : group;  
        final ConsumerFactory<?, ?> consumerFactory = createKafkaConsumerFactory(anonymous, consumerGroup,  
                extendedConsumerProperties);  
        int partitionCount = extendedConsumerProperties.getInstanceCount()  
                * extendedConsumerProperties.getConcurrency();  
        //如果AutoRebalanceEnabled=false的话，当instanceCount*Concurrency比实际topic的partitions数量还要多的话，将报错，因为会启动空闲的consumer  
        Collection<PartitionInfo> allPartitions = provisioningProvider.getPartitionsForTopic(partitionCount,  
                extendedConsumerProperties.getExtension().isAutoRebalanceEnabled(),  
                new Callable<Collection<PartitionInfo>>() {  
  
                    @Override  
                    public Collection<PartitionInfo> call() throws Exception {  
                        Consumer<?, ?> consumer = consumerFactory.createConsumer();  
                        List<PartitionInfo> partitionsFor = consumer.partitionsFor(destination.getName());  
                        consumer.close();  
                        return partitionsFor;  
                    }  
  
                });  
  
        Collection<PartitionInfo> listenedPartitions;  
  
        if (extendedConsumerProperties.getExtension().isAutoRebalanceEnabled() ||  
                extendedConsumerProperties.getInstanceCount() == 1) {  
            listenedPartitions = allPartitions;  
        }  
        else {  
            listenedPartitions = new ArrayList<>();  
            //为该Instance计算应该分派哪些partitions  
            for (PartitionInfo partition : allPartitions) {  
                // divide partitions across modules  
                if ((partition.partition()  
                        % extendedConsumerProperties.getInstanceCount()) == extendedConsumerProperties  
                                .getInstanceIndex()) {  
                    listenedPartitions.add(partition);  
                }  
            }  
        }  
        this.topicsInUse.put(destination.getName(), new TopicInformation(group, listenedPartitions));  
  
        Assert.isTrue(!CollectionUtils.isEmpty(listenedPartitions), "A list of partitions must be provided");  
        final TopicPartitionInitialOffset[] topicPartitionInitialOffsets = getTopicPartitionInitialOffsets(  
                listenedPartitions);  
        final ContainerProperties containerProperties = anonymous  
                || extendedConsumerProperties.getExtension().isAutoRebalanceEnabled()  
                        ? new ContainerProperties(destination.getName())  
                        : new ContainerProperties(topicPartitionInitialOffsets);  
        //并行度取consumer参数中设置的Concurrency和实际处理的partitions子集中较小的一个。  
        int concurrency = Math.min(extendedConsumerProperties.getConcurrency(), listenedPartitions.size());  
        @SuppressWarnings("rawtypes")  
        //ConcurrentMessageListenerContainer的doStart方法将根据并行度创建consumer  
        final ConcurrentMessageListenerContainer<?, ?> messageListenerContainer =  
                new ConcurrentMessageListenerContainer(  
                        consumerFactory, containerProperties) {  
  
            @Override  
            public void stop(Runnable callback) {  
                super.stop(callback);  
            }  
  
        };  
        messageListenerContainer.setConcurrency(concurrency);  
        if (!extendedConsumerProperties.getExtension().isAutoCommitOffset()) {  
            messageListenerContainer.getContainerProperties()  
                    .setAckMode(AbstractMessageListenerContainer.AckMode.MANUAL);  
            messageListenerContainer.getContainerProperties().setAckOnError(false);  
        }  
        else {  
            messageListenerContainer.getContainerProperties()  
                    .setAckOnError(isAutoCommitOnError(extendedConsumerProperties));  
        }  
        if (this.logger.isDebugEnabled()) {  
            this.logger.debug(  
                    "Listened partitions: " + StringUtils.collectionToCommaDelimitedString(listenedPartitions));  
        }  
        final KafkaMessageDrivenChannelAdapter<?, ?> kafkaMessageDrivenChannelAdapter = new KafkaMessageDrivenChannelAdapter<>(  
                messageListenerContainer);  
        kafkaMessageDrivenChannelAdapter.setBeanFactory(this.getBeanFactory());  
        ErrorInfrastructure errorInfrastructure = registerErrorInfrastructure(destination, consumerGroup,  
                extendedConsumerProperties);  
        if (extendedConsumerProperties.getMaxAttempts() > 1) {  
            kafkaMessageDrivenChannelAdapter.setRetryTemplate(buildRetryTemplate(extendedConsumerProperties));  
            kafkaMessageDrivenChannelAdapter.setRecoveryCallback(errorInfrastructure.getRecoverer());  
        }  
        else {  
            kafkaMessageDrivenChannelAdapter.setErrorChannel(errorInfrastructure.getErrorChannel());  
        }  
        return kafkaMessageDrivenChannelAdapter;  
    }  
org.springframework.kafka.listener.ConcurrentMessageListenerContainer#doStart方法：


[java] view plain copy
@Override  
    protected void doStart() {  
        if (!isRunning()) {  
            ContainerProperties containerProperties = getContainerProperties();  
            TopicPartitionInitialOffset[] topicPartitions = containerProperties.getTopicPartitions();  
            if (topicPartitions != null  
                    && this.concurrency > topicPartitions.length) {  
                this.logger.warn("When specific partitions are provided, the concurrency must be less than or "  
                        + "equal to the number of partitions; reduced from " + this.concurrency + " to "  
                        + topicPartitions.length);  
                this.concurrency = topicPartitions.length;  
            }  
            setRunning(true);  
                        //根据并行度生成N个KafkaMessageListenerContainer，  
                        //每个KafkaMessageListenerContainer的构造函数中会初始化一个KafkaConsumer  
            for (int i = 0; i < this.concurrency; i++) {  
                KafkaMessageListenerContainer<K, V> container;  
                if (topicPartitions == null) {  
                    container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties);  
                }  
                else {  
                //从前面分给Instance的partitions子集中，再指派一些partitions给具体的并行consumer  
                    container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties,  
                            partitionSubset(containerProperties, i));  
                }  
                if (getBeanName() != null) {  
                    container.setBeanName(getBeanName() + "-" + i);  
                }  
                if (getApplicationEventPublisher() != null) {  
                    container.setApplicationEventPublisher(getApplicationEventPublisher());  
                }  
                container.setClientIdSuffix("-" + i);  
                container.start();  
                this.containers.add(container);  
            }  
        }  
    }  
  
  //具体的二级子集生成策略  
    private TopicPartitionInitialOffset[] partitionSubset(ContainerProperties containerProperties, int i) {  
        TopicPartitionInitialOffset[] topicPartitions = containerProperties.getTopicPartitions();  
        if (this.concurrency == 1) {  
            return topicPartitions;  
        }  
        else {  
            int numPartitions = topicPartitions.length;  
            if (numPartitions == this.concurrency) {  
                return new TopicPartitionInitialOffset[] { topicPartitions[i] };  
            }  
            else {  
                int perContainer = numPartitions / this.concurrency;  
                TopicPartitionInitialOffset[] subset;  
                if (i == this.concurrency - 1) {  
                    subset = Arrays.copyOfRange(topicPartitions, i * perContainer, topicPartitions.length);  
                }  
                else {  
                    subset = Arrays.copyOfRange(topicPartitions, i * perContainer, (i + 1) * perContainer);  
                }  
                return subset;  
            }  
        }  
    }  
看完以上核心代码，应该就很清楚partitions的分配策略了，下面我们改一下场景，启动2个instance,每个instance的concurrency=3，那么：

instance0将分到[p0,p2,p4,p6]这4个分区，然后会启动3个consumer,consumer0将消费[p0],consumer1消费[p2],consumer2消费[p4,p6]。

instance0将分到[p1,p3,p5,p7]这4个分区，然后会启动3个consumer,consumer0将消费[p1],consumer1消费[p3],consumer2消费[p5,p7]。

可以看到每个instance的consumer2其实会多消费一个分区，不是很均匀，所以应该尽可能能整除，比如启动2个instance，concurrency=4，或者concurrency=2。当concurrency=2时，每个实例的每个consumer将消费两个partition。

看到这里，你应该对这几个参数有了一个比较深刻的理解了，这样在stream程序扩容或者缩容的时候应该能正确的配置参数.首先，kafka的topic是由多个partitions物理分隔的。假设topic: testIn,有8个partitions

其次，我们编写的springcloud kafka stream程序，打成jar包后，可以部署多个不同的实例instances，假设部署了3个instance。

那么这3个instance是怎么分配这8个partitions的呢？

在spring.cloud.stream.kafka.bindings.input.consumer.autoRebalanceEnabled=true时(默认)，多个实例会自动均衡

的分配partitions,由ConsumerCoordinator来自动协调，不用您操心了（跟instanceCount 和instanceIndex 没关系了，但是concurrency还是同理的）， 当设置成false后：

主要是通过consumer的4个参数(org.springframework.cloud.stream.binder.ConsumerProperties)来决定的:



spring.cloud.stream.bindings.input.consumer.partitioned=true

准备启动多少个实例

spring.cloud.stream.bindings.input.consumer.instanceCount =3

该实例编号index,从0开始到instanceCount -1

spring.cloud.stream.bindings.input.consumer.instanceIndex =0

每个实例中启动多少个kafka consumer

spring.cloud.stream.bindings.input.consumer.concurrency  =1



如果按照上面的配置的话(3个instance,每个instance的并行度1):

instance0将启动一个consumer0,消费p0,p3,p6; 

instance1将启动一个consumer0,消费p1,p4,p7; 

instance2将启动一个consumer0,消费p2,p5;

主要逻辑在org.springframework.cloud.stream.binder.kafka.KafkaMessageChannelBinder#createConsumerEndpoint这个方法中:

[java] view plain copy
protected MessageProducer createConsumerEndpoint(final ConsumerDestination destination, final String group,  
            final ExtendedConsumerProperties<KafkaConsumerProperties> extendedConsumerProperties) {  
  
        boolean anonymous = !StringUtils.hasText(group);  
        Assert.isTrue(!anonymous || !extendedConsumerProperties.getExtension().isEnableDlq(),  
                "DLQ support is not available for anonymous subscriptions");  
        String consumerGroup = anonymous ? "anonymous." + UUID.randomUUID().toString() : group;  
        final ConsumerFactory<?, ?> consumerFactory = createKafkaConsumerFactory(anonymous, consumerGroup,  
                extendedConsumerProperties);  
        int partitionCount = extendedConsumerProperties.getInstanceCount()  
                * extendedConsumerProperties.getConcurrency();  
        //如果AutoRebalanceEnabled=false的话，当instanceCount*Concurrency比实际topic的partitions数量还要多的话，将报错，因为会启动空闲的consumer  
        Collection<PartitionInfo> allPartitions = provisioningProvider.getPartitionsForTopic(partitionCount,  
                extendedConsumerProperties.getExtension().isAutoRebalanceEnabled(),  
                new Callable<Collection<PartitionInfo>>() {  
  
                    @Override  
                    public Collection<PartitionInfo> call() throws Exception {  
                        Consumer<?, ?> consumer = consumerFactory.createConsumer();  
                        List<PartitionInfo> partitionsFor = consumer.partitionsFor(destination.getName());  
                        consumer.close();  
                        return partitionsFor;  
                    }  
  
                });  
  
        Collection<PartitionInfo> listenedPartitions;  
  
        if (extendedConsumerProperties.getExtension().isAutoRebalanceEnabled() ||  
                extendedConsumerProperties.getInstanceCount() == 1) {  
            listenedPartitions = allPartitions;  
        }  
        else {  
            listenedPartitions = new ArrayList<>();  
            //为该Instance计算应该分派哪些partitions  
            for (PartitionInfo partition : allPartitions) {  
                // divide partitions across modules  
                if ((partition.partition()  
                        % extendedConsumerProperties.getInstanceCount()) == extendedConsumerProperties  
                                .getInstanceIndex()) {  
                    listenedPartitions.add(partition);  
                }  
            }  
        }  
        this.topicsInUse.put(destination.getName(), new TopicInformation(group, listenedPartitions));  
  
        Assert.isTrue(!CollectionUtils.isEmpty(listenedPartitions), "A list of partitions must be provided");  
        final TopicPartitionInitialOffset[] topicPartitionInitialOffsets = getTopicPartitionInitialOffsets(  
                listenedPartitions);  
        final ContainerProperties containerProperties = anonymous  
                || extendedConsumerProperties.getExtension().isAutoRebalanceEnabled()  
                        ? new ContainerProperties(destination.getName())  
                        : new ContainerProperties(topicPartitionInitialOffsets);  
        //并行度取consumer参数中设置的Concurrency和实际处理的partitions子集中较小的一个。  
        int concurrency = Math.min(extendedConsumerProperties.getConcurrency(), listenedPartitions.size());  
        @SuppressWarnings("rawtypes")  
        //ConcurrentMessageListenerContainer的doStart方法将根据并行度创建consumer  
        final ConcurrentMessageListenerContainer<?, ?> messageListenerContainer =  
                new ConcurrentMessageListenerContainer(  
                        consumerFactory, containerProperties) {  
  
            @Override  
            public void stop(Runnable callback) {  
                super.stop(callback);  
            }  
  
        };  
        messageListenerContainer.setConcurrency(concurrency);  
        if (!extendedConsumerProperties.getExtension().isAutoCommitOffset()) {  
            messageListenerContainer.getContainerProperties()  
                    .setAckMode(AbstractMessageListenerContainer.AckMode.MANUAL);  
            messageListenerContainer.getContainerProperties().setAckOnError(false);  
        }  
        else {  
            messageListenerContainer.getContainerProperties()  
                    .setAckOnError(isAutoCommitOnError(extendedConsumerProperties));  
        }  
        if (this.logger.isDebugEnabled()) {  
            this.logger.debug(  
                    "Listened partitions: " + StringUtils.collectionToCommaDelimitedString(listenedPartitions));  
        }  
        final KafkaMessageDrivenChannelAdapter<?, ?> kafkaMessageDrivenChannelAdapter = new KafkaMessageDrivenChannelAdapter<>(  
                messageListenerContainer);  
        kafkaMessageDrivenChannelAdapter.setBeanFactory(this.getBeanFactory());  
        ErrorInfrastructure errorInfrastructure = registerErrorInfrastructure(destination, consumerGroup,  
                extendedConsumerProperties);  
        if (extendedConsumerProperties.getMaxAttempts() > 1) {  
            kafkaMessageDrivenChannelAdapter.setRetryTemplate(buildRetryTemplate(extendedConsumerProperties));  
            kafkaMessageDrivenChannelAdapter.setRecoveryCallback(errorInfrastructure.getRecoverer());  
        }  
        else {  
            kafkaMessageDrivenChannelAdapter.setErrorChannel(errorInfrastructure.getErrorChannel());  
        }  
        return kafkaMessageDrivenChannelAdapter;  
    }  
org.springframework.kafka.listener.ConcurrentMessageListenerContainer#doStart方法：


[java] view plain copy
@Override  
    protected void doStart() {  
        if (!isRunning()) {  
            ContainerProperties containerProperties = getContainerProperties();  
            TopicPartitionInitialOffset[] topicPartitions = containerProperties.getTopicPartitions();  
            if (topicPartitions != null  
                    && this.concurrency > topicPartitions.length) {  
                this.logger.warn("When specific partitions are provided, the concurrency must be less than or "  
                        + "equal to the number of partitions; reduced from " + this.concurrency + " to "  
                        + topicPartitions.length);  
                this.concurrency = topicPartitions.length;  
            }  
            setRunning(true);  
                        //根据并行度生成N个KafkaMessageListenerContainer，  
                        //每个KafkaMessageListenerContainer的构造函数中会初始化一个KafkaConsumer  
            for (int i = 0; i < this.concurrency; i++) {  
                KafkaMessageListenerContainer<K, V> container;  
                if (topicPartitions == null) {  
                    container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties);  
                }  
                else {  
                //从前面分给Instance的partitions子集中，再指派一些partitions给具体的并行consumer  
                    container = new KafkaMessageListenerContainer<>(this.consumerFactory, containerProperties,  
                            partitionSubset(containerProperties, i));  
                }  
                if (getBeanName() != null) {  
                    container.setBeanName(getBeanName() + "-" + i);  
                }  
                if (getApplicationEventPublisher() != null) {  
                    container.setApplicationEventPublisher(getApplicationEventPublisher());  
                }  
                container.setClientIdSuffix("-" + i);  
                container.start();  
                this.containers.add(container);  
            }  
        }  
    }  
  
  //具体的二级子集生成策略  
    private TopicPartitionInitialOffset[] partitionSubset(ContainerProperties containerProperties, int i) {  
        TopicPartitionInitialOffset[] topicPartitions = containerProperties.getTopicPartitions();  
        if (this.concurrency == 1) {  
            return topicPartitions;  
        }  
        else {  
            int numPartitions = topicPartitions.length;  
            if (numPartitions == this.concurrency) {  
                return new TopicPartitionInitialOffset[] { topicPartitions[i] };  
            }  
            else {  
                int perContainer = numPartitions / this.concurrency;  
                TopicPartitionInitialOffset[] subset;  
                if (i == this.concurrency - 1) {  
                    subset = Arrays.copyOfRange(topicPartitions, i * perContainer, topicPartitions.length);  
                }  
                else {  
                    subset = Arrays.copyOfRange(topicPartitions, i * perContainer, (i + 1) * perContainer);  
                }  
                return subset;  
            }  
        }  
    }  
看完以上核心代码，应该就很清楚partitions的分配策略了，下面我们改一下场景，启动2个instance,每个instance的concurrency=3，那么：

instance0将分到[p0,p2,p4,p6]这4个分区，然后会启动3个consumer,consumer0将消费[p0],consumer1消费[p2],consumer2消费[p4,p6]。

instance0将分到[p1,p3,p5,p7]这4个分区，然后会启动3个consumer,consumer0将消费[p1],consumer1消费[p3],consumer2消费[p5,p7]。

可以看到每个instance的consumer2其实会多消费一个分区，不是很均匀，所以应该尽可能能整除，比如启动2个instance，concurrency=4，或者concurrency=2。当concurrency=2时，每个实例的每个consumer将消费两个partition。

看到这里，你应该对这几个参数有了一个比较深刻的理解了，这样在stream程序扩容或者缩容的时候应该能正确的配置参数.