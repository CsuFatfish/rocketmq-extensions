# RocketmqExtendTools

Since the official API provided by RocketMQ and RocketmqConsole only provide some basic features, in actual development, developers often need to expand based on the basic API. The purpose of this project is to expand its features based on RocketMQ, provide some easy-to-use APIs for features that are not directly implemented by the official, for everyone to use or learn from, reduce learning costs, and improve efficiency.

The features currently implemented are:

1. Message idempotent deduplication solution to prevent repeated consumption of messages.
2. The elastic scaling of Producer and Consumer automatically scales producers and consumers based on resource usage.
3. Realize some common features: obtain the logical position of the message in the queue, obtain the tps of production and consumption, obtain the corresponding relationship between the queue and its consumers, etc.



Since the current project does not consider distributed and some actual use scenarios, the project will be improved in the future, mainly in the following aspects:

1. The currently used idempotent deduplication scheme is a heavyweight scheme, and the problem of repeated message consumption is a low probability problem. For the processing of small probability problems, heavyweight schemes may not be used. Because there are caches or database operations before and after each message consumption, the first problem is the low efficiency, and the second problem is the consistency of redis and MySQL.

A lightweight solution will be added later. For a system, it has tolerance. For example, repeated messages will only appear within 5 minutes. Then you can directly use redis to prevent repeated consumption. Redis stores message information. The survival time is set to 6 minutes, so as long as the message can be found in redis, the current message is a repeated message, otherwise it is not.

2. The elastic scaling of Producer and Consumer only considers the elastic scaling of the local machine. RocketMQ is a distributed system. Both consumers and producers exist in the form of clusters. The elastic scaling should be designed to communicate through RPC or network. Scale producers and consumers in producer and consumer clusters.
3. Several commonly used features are implemented through the RocketMQ client in the rocketmq-tools package. Calling the API is actually performing network communication, that is, the obtained information is not real-time information, but the information of the previous snapshot status. Fetching real-time status will be considered later.



## Message idempotent deduplication

The implementation of message idempotent deduplication is in the duplicate package, which supports the use of redis or MySQL as idempotent tables, and also supports the use of redis and MySQL for double checking.



When using message idempotent deduplication, the startup configuration of Consumer is not much different from usual. The only difference is that the user needs to customize a class, let it inherit AvoidDuplicateMsgListener, and register it as the MessageListener of consumer.



### Principle flow chart

![image-20220621212235579](https://github.com/CsuFatfish/rocketmq-extend-tools/blob/main/Principle%20flow%20chart.jpg)



### Instructions for use

1. Create a custom class to inherit from AvoidDuplicateMsgListener. Note that the parameter of its constructor is DuplicateConfig, and the constructor of the parent class is called during construction. The doHandleMsg method rewritten by this class is the logic for actually consuming messages.

```java
public class DuplicateTest extends AvoidDuplicateMsgListener {

    // The constructor must be written in this way, call the constructor of the parent class, and the parameter is DuplicateConfig.
    public DuplicateTest(DuplicateConfig duplicateConfig) {
        super(duplicateConfig);
    }

    // The overridden method is the logic of consuming messages.
    @Override
    protected boolean doHandleMsg(MessageExt messageExt) {
        switch (messageExt.getTopic()) {
            case "DUPLICATE-TEST-TOPIC": {
                log.info("consuming..., messageId: {}", messageExt.getMsgId());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                break;
            }
        }

        return true;
    }
}
```

2. Depending on the use of redis or/and MySQL, use the uniqKey of the message or the business key (the business key refers to the key set by calling the setKeys method when sending the message) for deduplication, and call different enableAvoidDuplicateConsumeConfig methods to obtain DuplicateConfig.

```java
// Use redis and MySQL, specify the MySQL table name, and applicationName indicates which application to do deduplication for. The deduplication of the same message in different applications is processed in isolation, and msgKeyStrategy can be USERKEY_AS_MSG_KEY or UNIQKEY_AS_MSG_KEY.
public static DuplicateConfig enableAvoidDuplicateConsumeConfig(String applicationName, int msgKeyStrategy, StringRedisTemplate redisTemplate, JdbcTemplate jdbcTemplate, String tableName) {
    return new DuplicateConfig(applicationName, AVOID_DUPLICATE_ENABLE, msgKeyStrategy, redisTemplate, jdbcTemplate, tableName);
}

// use redis
public static DuplicateConfig enableAvoidDuplicateConsumeConfig(String applicationName, int msgKeyStrategy, StringRedisTemplate redisTemplate) {
    return new DuplicateConfig(applicationName, AVOID_DUPLICATE_ENABLE, msgKeyStrategy, redisTemplate);
}

// Use MySQL
public static DuplicateConfig enableAvoidDuplicateConsumeConfig(String applicationName, int msgKeyStrategy, JdbcTemplate jdbcTemplate, String tableName) {
    return new DuplicateConfig(applicationName, AVOID_DUPLICATE_ENABLE, msgKeyStrategy, jdbcTemplate, tableName);
}
```

Note that the columns of the MySQL idempotent table are fixed, and the table name can be customized. The table creation statement is:

```sql
CREATE TABLE `xxxxx` (
`application_name` varchar(255) NOT NULL COMMENT 'Consumption application name (consumer group name can be used)',
`topic` varchar(255) NOT NULL COMMENT 'The topic of the message source (different topic messages will not be considered duplicates)',
`tag` varchar(16) NOT NULL COMMENT 'The tag of the message (the same topic with different tags, even if the key is deduplicated, the same key will not be considered duplicate), if there is no tag, the "" string will be stored',
`msg_uniq_key` varchar(255) NOT NULL COMMENT 'The unique key of the message (it is recommended to use the business primary key)',
`status` varchar(16) NOT NULL COMMENT 'The consumption status of this message',
`expire_time` bigint(20) NOT NULL COMMENT 'The expiration time (timestamp) of this deduplication record',
UNIQUE KEY `uniq_key` (`application_name`,`topic`,`tag`,`msg_uniq_key`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=COMPACT;
```

3. New custom class, register it as the MessageListener of the consumer, and idempotent de-duplication will take effect. Here we use redis as an example. In other cases, the code can be found in the MainTest class of the duplicate package under the test package.

```java
@Test
public static void redisTest() throws MQClientException {
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("DUPLICATE-TEST-GROUP");
    consumer.subscribe("DUPLICATE-TEST-TOPIC", "*");
    // To specify the NameServer, the user needs to specify the NameServer address
    consumer.setNamesrvAddr("192.168.20.100:9876");

    consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

    String appName = consumer.getConsumerGroup();
    // User is required to provide StringRedisTemplate
    StringRedisTemplate stringRedisTemplate = xxx;
    // Call the enableAvoidDuplicateConsumeConfig method of DuplicateConfig to create DuplicateConfig
    DuplicateConfig duplicateConfig = DuplicateConfig.enableAvoidDuplicateConsumeConfig(appName, USERKEY_AS_MSG_KEY, stringRedisTemplate);
    // Create a user-defined Listener class
    AvoidDuplicateMsgListener msgListener = new DuplicateTest(duplicateConfig);
    // Register the Listener to the consumer, and idempotent deduplication will take effect
    consumer.registerMessageListener(msgListener);
    consumer.start();
}
```



## Elastic scaling of Producer and Consumer

The elastic scaling of Producer and Consumer is implemented in the scaling package, which supports optional monitoring of cpu, memory, produce tps, and consume tps indicators, and elastically expands or reduces the number of Producers or Consumers based on this.



### Instructions for use

#### Consumer Elastic Scaling

1. To create a consumer startup template, the user needs to provide the consumer group name, NameServer address, ConsumeFromWhere, subscribed topic and tag, consumption mode (cluster or broadcast), and MessageListenerConcurrently.

```java
Map<String, String> topicAndTag = new HashMap<>();
topicAndTag.put("SCALE", "*");
// The parameters are: consumer group name, NameServer address, ConsumeFromWhere, subscribed topic and tag (saved with Map), consumption mode (cluster or broadcast), MessageListenerConcurrently
ConsumerStartingTemplate consumerStartingTemplate = new ConsumerStartingTemplate("SCALE_TEST_CONSUMER_GROUP", "192.168.20.100:9876", ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET,
        topicAndTag, MessageModel.CLUSTERING, new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        // Consume messages one by one
        for (MessageExt msg : msgs) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        // Return consumption status: Consumption successful
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
```

2. To create a consumer scaling group, the user needs to provide the scaling group name, the minimum number of consumer instances, the maximum number of consumer instances, the cooling time (that is, the shortest time interval between two consecutive scalings), the consumer startup template, and the shrinking strategy (remove The latest or earliest created consumer, ConsumerRemoveStrategy provides getOldestRemoveStrategy() and getNewestRemoveStrategy() methods).

```java
// The parameters are the name of the scaling group, the minimum number of consumer instances, the maximum number of consumer instances, the cooling time (that is, the shortest time interval between two consecutive scalings, in seconds), the consumer startup template, and the shrinking strategy (remove the earliest or latest created consumer)
ConsumerScalingGroup consumerScalingGroup = new ConsumerScalingGroup("CONSUMER-SCALE", 0, 3, 5, consumerStartingTemplate, new ConsumerRemoveStrategy().getOldestRemoveStrategy());
```

3. Create a monitor class (there are four monitors: CpuMonitorRunnable, MemoryMonitorRunnable, ProduceTpsMonitor, ConsumeTpsMonitor, which monitor cpu, memory, production tps, and consumption tps respectively), requiring the user to provide the monitor name, scaling group name, scaling group, and scheduled execution The cron expression, statistical duration (unit: second), statistical method (maximum, minimum, average value), expansion threshold, shrinking threshold, how many times the threshold is reached consecutively to trigger scaling. In addition, if it is production and consumption tps monitoring, brokerAddr is also required.

```java
// The parameters are monitor name, scaling group name, scaling group, cron expression for timing execution, statistical duration (unit: second), statistical method (maximum, minimum, average value), expansion threshold, shrinking threshold, the number of times the threshold needs to be reached consecutively when scaling is triggered.
CpuMonitorRunnable cpuMonitorRunnable = new CpuMonitorRunnable("CPU-CONSUMER-MONITER", "CONSUMER-SCALE", consumerScalingGroup, "0/5 * * * * ?", 3, MonitorMethod.AVG, 80, 20, 3);

// For production and consumption tps monitoring, brokerAddr is also required.
ProduceTpsMonitor produceTpsMonitor = new ProduceTpsMonitor("PRODUCE_TPS_MONITOR", "PRODUCER_SCALING_GROUP", producerScalingGroup, "0/5 * * * * ?", 3, MonitorMethod.AVG, 0, -1, 3, "192.168.20.100:10911");
```

4. Create a MonitorManager, and call its startMonitor method to start the monitor created in step 3, and auto scaling starts.

```java
MonitorManager monitorManager = new MonitorManager();
monitorManager.startMonitor(cpuMonitorRunnable);
```

5. The entire code for enabling consumer elastic scaling is as follows. You can also view the ConsumerScalingTest class in the scaling package under the test package.

```java
    @Test
    public void testScalingConsumer() {
        Map<String, String> topicAndTag = new HashMap<>();
        topicAndTag.put("SCALE", "*");
        // Create a consumer launch template
        ConsumerStartingTemplate consumerStartingTemplate = new ConsumerStartingTemplate("SCALE_TEST_CONSUMER_GROUP", "192.168.20.100:9876", ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET,
                topicAndTag, MessageModel.CLUSTERING, new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                // Consume messages one by one
                for (MessageExt msg : msgs) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                // Return consumption status: Consumption successful
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // Create a consumer scaling group
        ConsumerScalingGroup consumerScalingGroup = new ConsumerScalingGroup("CONSUMER-SCALE", 0, 3, 5,
                                                        consumerStartingTemplate, new ConsumerRemoveStrategy().getOldestRemoveStrategy());
        // Create a resource monitor
        CpuMonitorRunnable cpuMonitorRunnable = new CpuMonitorRunnable("CPU-CONSUMER-MONITER", "CONSUMER-SCALE", consumerScalingGroup,
                                                    "0/5 * * * * ?", 3, MonitorMethod.AVG, 80, 20, 3);
        // This is just for testing under the windows system, assuming that the cpu utilization is always 90%.
        cpuMonitorRunnable.setOs("windows");
        cpuMonitorRunnable.setCmd("echo 90");
//        cpuMonitorRunnable.setCmd("echo 10");
        // Create MonitorManager and start monitor
        MonitorManager monitorManager = new MonitorManager();
        monitorManager.startMonitor(cpuMonitorRunnable);

        // while true is to avoid the end of the program, in order to print out the log information.
        while (true) {

        }
    }
```

#### Producer Elastic Scaling

1. To create a producer startup template, the user needs to provide the producer group name, NameServer address, number of retries after sending failures, and how many queues a topic corresponds to.

```java
// The parameters are producer group name, NameServer address, number of retries after sending failure, and how many Queues correspond to a topic.
ProducerStartingTemplate producerStartingTemplate = new ProducerStartingTemplate("SCALE_PRODUCER_GROUP", "192.168.20.100:9876", 0, 2);
```

2. To create a producer scaling group, the user needs to provide the scaling group name, minimum number of producer instances, maximum number of producer instances, cooling time, producer startup template, and shrinking strategy (remove the latest or earliest created producer, ProducerRemoveStrategy provides the getOldestRemoveStrategy() and getNewestRemoveStrategy() methods).

```java
// The parameters are scaling group name, minimum number of producer instances, maximum number of producer instances, cooling time, producer startup template, and shrinking policy.
ProducerScalingGroup producerScalingGroup = new ProducerScalingGroup("PRODUCER_SCALING_GROUP", 0, 3, 5, producerStartingTemplate, new ProducerRemoveStrategy().getOldestRemoveStrategy());
```

3. Create a monitor class. Here we take the production tps monitor as an example. The user needs to provide the monitor name, scaling group name, scaling group, cron expression for timing execution, statistical duration (unit: second), and statistical method (maximum , minimum, average value), expansion threshold, shrinking threshold, how many times the threshold is reached consecutively to trigger scaling, and brokerAddr.

```java
// The parameters are monitor name, scaling group name, scaling group, cron expression for timing execution, statistical duration (unit: second), statistical method (maximum, minimum, average value), expansion threshold, shrinking threshold, and continuous reaching threshold How many times to trigger scaling, brokerAddr.
ProduceTpsMonitor produceTpsMonitor = new ProduceTpsMonitor("PRODUCE_TPS_MONITOR", "PRODUCER_SCALING_GROUP", producerScalingGroup, "0/5 * * * * ?", 3, MonitorMethod.AVG, 0, -1, 3, "192.168.20.100:10911");
```

4. Create a MonitorManager and start the Monitor, and auto scaling starts.

```java
MonitorManager monitorManager = new MonitorManager();
monitorManager.startMonitor(produceTpsMonitor);
```

5. The entire code for enabling producer auto scaling is as follows. You can also view the ProducerScalingTest class in the scaling package under the test package.

```java
@Test
public void testScalingProducer() {
    //Create a producer launch template
    ProducerStartingTemplate producerStartingTemplate = new ProducerStartingTemplate("SCALE_PRODUCER_GROUP",
            "192.168.20.100:9876", 0, 2);
    // Create a producer scaling group
    ProducerScalingGroup producerScalingGroup = new ProducerScalingGroup("PRODUCER_SCALING_GROUP", 0, 3, 5, producerStartingTemplate, new ProducerRemoveStrategy().getOldestRemoveStrategy());
    // Create a resource monitor
    ProduceTpsMonitor produceTpsMonitor = new ProduceTpsMonitor("PRODUCE_TPS_MONITOR", "PRODUCER_SCALING_GROUP", producerScalingGroup,
            "0/5 * * * * ?", 3, MonitorMethod.AVG, 0, -1, 3, "192.168.20.100:10911");
    // Create MonitorManager and start Monitor
    MonitorManager monitorManager = new MonitorManager();
    monitorManager.startMonitor(produceTpsMonitor);

    // while true is to avoid the end of the program, in order to print out the log information.
    while (true) {

    }
}
```



## Other features

Other features are implemented in the service package, and the entry is the impl package, which provides BrokerServiceImpl, MessageServiceImpl, QueueServiceImpl, and TopicServiceImpl. The following three features are listed as examples.



### Get the logical position of the message in the queue

```java
// Create MessageServiceImpl
MessageServiceImpl messageService = new MessageServiceImpl();
// After the producer sends a successful message, use SendResult as a parameter and call the getMessageLogicPositionInQueueAfterSend method to get the logical position of the message in the queue.
Long logicPos = messageService.getMessageLogicPositionInQueueAfterSend(sendResult);
```



### Get production and consumption tps

```java
// Create BrokerServiceImpl
BrokerServiceImpl brokerServiceImpl = new BrokerServiceImpl();
// Call the getProduceAndConsumeTpsByBrokerAddr(String brokerAddr) method to obtain the tps of production and consumption, and the return value is a Map. The value corresponding to the key "produceTps" is the production tps, and the value corresponding to the key "consumeTps" is the consumption tps.
Map<String, Long> map = brokerServiceImpl.getProduceAndConsumeTpsByBrokerAddr("192.168.20.100:10911");
Long produceTps = map.get("produceTps");
Long consumeTps = map.get("consumeTps");
```



### Get the corresponding relationship between the queue and its consumers

```java
// Create QueueServiceImpl
QueueServiceImpl queueServiceImpl = new QueueServiceImpl();
// Call the getQueueConsumerRelationByConsumerGroup(String consumerGroupName) method to obtain the corresponding relationship between the queue and its consumers through the consumer group name, and the return value is Map<MessageQueue, String>.
queueServiceImpl.getQueueConsumerRelationByConsumerGroup(xxx);
```



## TODO List

1. For idempotent deduplication, add a lightweight solution. For a system, it has tolerance. For example, repeated messages will only appear within 5 minutes, so you can directly use redis to prevent repeated consumption. Redis stores message information, and the survival time is set to 6 minutes. As long as the message can be found in redis, the current message is a repeated message, otherwise it is not.

2. Design elastic scaling to scale producers and consumers in producer and consumer clusters through RPC communication or network communication.

3. Several commonly used features are implemented through the RocketMQ client in the rocketmq-tools package. Calling the API is actually performing network communication, that is, the obtained information is not real-time information, but the information of the previous snapshot status. Fetching real-time status will be considered later.
