![](/assets/springcloudstreamkafkalistenerstacktrace.png)
![](/assets/springcloudstreamkafkalistenerstacktrace2.png)
![](/assets/springcloudstreamkafkalistenerstacktrace3.png)

部分日志：
`
2018-05-24 09:48:58.178 DEBUG 12248 --- [           main] o.a.tomcat.util.modeler.BaseModelMBean   : preRegister org.apache.tomcat.util.net.NioEndpoint@3f4964f2 Tomcat:type=ThreadPool,name="http-nio-8912"
2018-05-24 09:48:58.196 DEBUG 12248 --- [           -L-1] o.s.integration.channel.DirectChannel    : preSend on channel 'input', message: GenericMessage [payload=wangxinhetest, headers={kafka_offset=0, id=31610fff-aab9-1f59-7c15-5109b6839e78, kafka_receivedPartitionId=1, contentType=text/plain, kafka_receivedTopic=kafkatopics, timestamp=1527126538196}]
2018-05-24 09:48:58.197 DEBUG 12248 --- [           -L-1] o.s.c.s.b.StreamListenerMessageHandler   : org.springframework.cloud.stream.binding.StreamListenerMessageHandler@1788cb61 received message: GenericMessage [payload=wangxinhetest, headers={kafka_offset=0, id=31610fff-aab9-1f59-7c15-5109b6839e78, kafka_receivedPartitionId=1, contentType=text/plain, kafka_receivedTopic=kafkatopics, timestamp=1527126538196}]
2018-05-24 09:48:58.410 DEBUG 12248 --- [           -C-1] org.apache.kafka.clients.NetworkClient   : Sending metadata request {topics=[kafkatopics]} to node -2
2018-05-24 09:48:58.412 DEBUG 12248 --- [           -C-1] org.apache.kafka.clients.Metadata        : Updated cluster metadata version 2 to Cluster(id = 9qRK5JLrS8aewdHkWU26-g, nodes = [wxh-pc:9092 (id: 0 rack: null), wxh-pc:9094 (id: 2 rack: null), wxh-pc:9093 (id: 1 rack: null)], partitions = [Partition(topic = kafkatopics, partition = 0, leader = 0, replicas = [0,], isr = [0,]), Partition(topic = kafkatopics, partition = 1, leader = 1, replicas = [1,], isr = [1,]), Partition(topic = kafkatopics, partition = 2, leader = 2, replicas = [2,], isr = [2,]), Partition(topic = kafkatopics, partition = 3, leader = 0, replicas = [0,], isr = [0,])])
